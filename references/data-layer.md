# Data layer: Room, DataStore, network, DTOs, mappers, paging, sync

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

## Models per layer

```
Room XxxEntity / XxxTuple ──toDomain()──► domain Xxx ──toUi()──► XxxUi (@Immutable, pre-formatted strings)
     ▲                                        │
     └──── upsertFromServer(XxxResponseDto)   └── create/update params (BigDecimal, String?)
XxxEntity ──toRequest()──► XxxRequestDto ──► ApiService
```

<a id="types"></a>
### Types per layer

| Concept | Room / DTO | Domain | Ui |
|---|---|---|---|
| Money | `Double` (+ `ROUND(…, 2)` in SQL) | `BigDecimal` | `BigDecimal` + `amountString` (`toCustomPrice()`) |
| Time | epoch millis `Long` (`created_at`) | `Instant` | `String` (`toFormatDateTime()` / `toDateTime()`) |
| Server time | ISO `String` in DTO | `Instant.parse(...)` | — |
| Booleans | `Boolean` (`is_synced` INTEGER) | `Boolean` | `Boolean` |
| Enums in prefs | `String` (`name`) / `Int` (value class) | enum / value class | enum |

- Money: legacy — `DebtEntity.amount: Double` and `DebtRequestDto.amount: Double` (rounding risk is contained by
  `ROUND(…, 2)` in aggregate queries and `MAXIMUM_FRACTION_DIGITS = 2` in UI). **[proposal]** new money columns are
  `Long` minor units (cents) in Room and a decimal string in DTOs; domain stays `BigDecimal`.
- Mappers are top-level `internal fun` extensions in `mapper/` of the **target** layer: `data/mapper/NoteMapper.kt`
  (`NoteEntity.toDomain()`, `NoteEntity.toRequest()`), `presentation/mapper/CommonMapper.kt` (`Note.toUi()`).
- Derived business values are computed in the mapper that crosses into domain: `UserResponse.toDomain()` computes
  `isPro`/`isExpired` against `meta.serverTime` (server clock, not device clock).

---

<a id="room"></a>
## Room

`data/.../local/database/entity/DebtEntity.kt`
```kotlin
@Entity(
    tableName = "debts",
    indices = [
        Index(value = ["server_id"], unique = true),
        Index(value = ["server_update_seq"], unique = true),
        Index(
            value = ["contact_id", "created_at", "amount"],
            orders = [Index.Order.ASC, Index.Order.DESC, Index.Order.ASC]
        ),
        Index(value = ["is_synced"]),
    ],
    foreignKeys = [
        ForeignKey(entity = ContactEntity::class, parentColumns = ["id"], childColumns = ["contact_id"],
            onDelete = ForeignKey.CASCADE)
    ]
)
internal data class DebtEntity(
    @PrimaryKey(true) val id: Long,
    @ColumnInfo("server_id") val serverId: Long?,
    @ColumnInfo("server_update_seq") val serverUpdateSeq: Long?,
    @ColumnInfo("is_synced") val isSynced: Boolean,
    @ColumnInfo("contact_id") val contactId: Long,
    val amount: Double,
    val desc: String?,
    @ColumnInfo("created_at") val createdAt: Long,
    @ColumnInfo("updated_at") val updatedAt: Long
)
```
Rules:
- Table names plural `snake_case`; columns `snake_case` via `@ColumnInfo`; Kotlin properties `camelCase`.
- *(sync)* Every synced table has `server_id` (unique, nullable until first sync), `server_update_seq` (unique), `is_synced` (indexed),
  `created_at`, `updated_at`.
- An index for every real `WHERE` + `ORDER BY` (the composite debts index serves "debts of a contact by date" and the
  history balance `contact_id = ? AND created_at <= ?`).
- Children use `ForeignKey(... onDelete = CASCADE)`.
- Aggregates (balance, last transaction) are **computed in SQL** (`COALESCE(SUM(d.amount), 0)`, `MAX(d.created_at)`) and
  read into a `XxxTuple`, never stored. [hypothesis: no denormalized balance → no drift between debts and contact]
- Search: an FTS4 shadow table (`@Fts4(contentEntity = NoteEntity::class, tokenizer = FtsOptions.TOKENIZER_UNICODE61)`)
  queried with `MATCH ? || '*'` (prefix search, Unicode-aware for Cyrillic). [hypothesis: no search library needed]
- Database: `Room.databaseBuilder<AppDatabase>(context, DB_NAME)`. Target (skeleton): `exportSchema = true` with
  `ksp { arg("room.schemaLocation", "$projectDir/schemas") }`, schemas in VCS, an explicit `Migration` for every version
  bump — never `fallbackToDestructiveMigration` (user data would be lost). (legacy: reference has `exportSchema = false`.)

### DAO
- `internal interface XxxDao` with `@DaoReturnTypeConverters(PagingSourceDaoReturnTypeConverter::class)` (Room 3 paging).
- Observation returns `Flow<X?>` / `PagingSource<Int, X>`; one-shot is `suspend`.
- Partial updates are explicit `@Query("UPDATE … SET …, is_synced = 0 WHERE id = :id")` — every local edit marks the row
  dirty. `@Update` with full entities is used only by sync.
- Multi-statement operations are `@Transaction` default methods in the DAO (`deleteWithSync`, `upsertFromServer`).
- Dynamic queries (filters/sort/search) use `@RawQuery(observedEntities = [...])` with `RoomRawQuery`:

`data/.../repository/NoteRepositoryImpl.kt`
```kotlin
private fun pagingConstructQuery(searchQuery: String?): RoomRawQuery {
    val order: String
    var statementSearchQuery: String? = null

    var sql = if (searchQuery.isNullOrBlank()) {
        order = "updated_at DESC"
        "SELECT * FROM notes"
    } else {
        order = "n.updated_at DESC"
        statementSearchQuery = searchQuery
        """
            SELECT n.* FROM notes AS n
            JOIN notes_fts AS nf ON nf.rowid = n.id
            WHERE notes_fts MATCH ? || '*'
        """.trimIndent()
    }

    sql += " ORDER BY $order"            // fragments only from code / enums

    return RoomRawQuery(
        sql = sql,
        onBindStatement = { statement -> statementSearchQuery?.let { statement.bindText(1, it) } }   // user input is bound
    )
}
```
`ContactRepositoryImpl.pagingConstructQuery(filter)` builds `JOIN`/`WHERE`/`GROUP BY`/`HAVING`/`ORDER BY` from
`ContactType` and `ContactsSort` enums the same way.

### Transactions
Domain opens them via `TransactionRunner.withWriteTransaction { }` → `RoomTransactionRunner` → `appDatabase.withWriteTransaction`.
DAO-level multi-step writes are `@Transaction` methods. Logout clears three tables in one transaction (`LogoutUseCase`).

---

## DataStore

- One Preferences file `"settings"`: `internal val Context.dataStore by preferencesDataStore(name = "settings")`,
  provided as a `@Singleton DataStore<Preferences>`.
- Each owner class declares its keys as `private val xxxKey = …PreferencesKey("snake_case")`
  (`AppSettingsRepositoryImpl`, `UserDataStore`, `AppConfigRepositoryImpl`, `TokenGatewayImpl`).
- `observe()` = `dataStore.data.map { prefs -> Model(field = prefs[key] ?: Model.Default.field, …) }.distinctUntilChanged()`.
- Writes: one `suspend fun updateXxx(value)` per field, `dataStore.edit { it[key] = value }`; nullable fields remove the key.
- legacy: `SyncWorker` defines `is_fcm_synced`/`installation_id` keys in its own companion — two owners of one key.
  New keys have exactly one owner.

---

<a id="network"></a>
## Network (Retrofit + kotlinx.serialization)

`data/.../di/DataModule.kt`
```kotlin
@[Provides Singleton]
fun provideOkHttpClient(authInterceptor: AuthInterceptor): OkHttpClient =
    OkHttpClient.Builder().addInterceptor(authInterceptor).build()

@[Provides Singleton]
fun provideApiService(okHttpClient: OkHttpClient): ApiService {
    val json = Json { ignoreUnknownKeys = true }
    return Retrofit.Builder()
        .baseUrl(ApiFactory.BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
        .create(ApiService::class.java)
}
```
- One `internal interface ApiService`; every method `suspend fun x(@Body request: XRequest): Response<XResponse>`
  (`Response<Unit>` for empty bodies). Paths are relative, kebab-case (`reset-password/by-key`, `devices/current`).
- `AuthInterceptor` adds `Accept: application/json` and `Authorization: Bearer <token>` when a token exists.
- `ignoreUnknownKeys = true` — the server may add fields without breaking old clients.
- DTOs: `@Serializable internal data class` with `@SerialName("snake_case")` for multi-word fields; nested response parts
  as nested classes (`UserResponse.UserData`, `UserResponse.UserMeta`).
- legacy: `forgotPassword(@Body request: JsonObject)` built with `buildJsonObject` — use a typed `ForgotPasswordRequest`.
- `ApiFactory` holds `BASE_URL` and `LIMIT_COUNT = 200` (sync batch). **[proposal]** move `BASE_URL` to
  `buildConfigField` per build type (see [security.md](security.md#7-configuration)).

### `safeApiCall`
`data/.../remote/Utils.kt`
```kotlin
internal inline fun <T> safeApiCall(block: () -> Response<T>): Result<T, NetworkError> = try {
    val response = block()
    if (response.isSuccessful) {
        val body = response.body()
        requireNotNull(body)
        Result.success(body)
    } else {
        Result.error(NetworkError.Network(response.code()))
    }
} catch (_: UnknownHostException) { Result.error(NetworkError.UnknownHost)
} catch (_: SocketTimeoutException) { Result.error(NetworkError.SocketTimeout)
} catch (_: ConnectException) { Result.error(NetworkError.Connect)
} catch (_: IOException) { Result.error(NetworkError.IO)
} catch (e: CancellationException) { throw e
} catch (_: Exception) { Result.error(NetworkError.Any) }
```
The repository then maps to its area error: `.mapError { LoginError.Network(it) }` and to a domain value: `.map { it.token }`.

---

<a id="paging"></a>
## Paging 3

```kotlin
override fun observePaged(filter: ContactsFilter): Flow<PagingData<Contact>> = Pager(
    config = PagingConfig(pageSize = 30, prefetchDistance = 10, initialLoadSize = 30),
    pagingSourceFactory = { contactDao.pagingSource(pagingConstructQuery(filter)) }
).flow.map { pagingData -> pagingData.map { it.toDomain() } }
```
- Paging only from Room (no `RemoteMediator`: sync is separate, Room invalidation refreshes the list).
- ViewModel: `flatMapLatest` on the filter + `cachedIn(viewModelScope)`; Screen: `collectAsLazyPagingItems()`.
- `PagingData` is allowed in domain interfaces (pure-Kotlin `paging-common`).
- legacy: `PagingConfig(30, 10, 30)` repeated in three repositories — extract `PAGE_SIZE`/`PREFETCH_DISTANCE` constants.

---

<a id="sync"></a>
## *(optional)* Offline-first sync protocol

Only for apps whose data lives on several devices. Templates: [templates/optional/sync](../templates/optional/sync).

Local write path (`NoteRepositoryImpl.create/update/delete`):
1. Insert/update with `is_synced = 0`, `updated_at = System.currentTimeMillis()`.
2. Delete: `deleteWithSync(ids)` = `INSERT OR IGNORE INTO note_deletes(server_id) SELECT server_id … WHERE server_id IS NOT NULL`
   + `DELETE` in one transaction (rows never synced need no tombstone).
3. The use case calls `syncRepository.sync()`.

Sync loop (`XxxRepositoryImpl.sync()`), batch = `ApiFactory.LIMIT_COUNT` (200):
```
do {
    unsynced   = dao.getUnsyncedXxx(LIMIT)            // is_synced = 0
    deletedIds = dao.getDeletedIds(LIMIT)             // tombstones
    clientSeq  = dao.getMaxUpdateSeq()                // last server change we have
    POST sync/xxx { xxx: unsynced.toRequest(), deleted_ids, confirmed_deleted_ids, client_update_seq }
    on success:
        mark sent rows is_synced = 1; remove sent tombstones
        upsertFromServer(each returned row)           // match by local_id, then server_id
        deleteByServerIds(server.deleted_ids); ack them next round (confirmed_deleted_ids)
        finished = no list hit the LIMIT and nothing to ack
    on error: finished = true (result returned)
} while (!finished)
```
- Order matters: contacts before debts (debt DTO carries the contact's `server_id`; debts of unsynced contacts are skipped by
  `getUnsyncedDebts` `JOIN contacts … server_id IS NOT NULL`).
- Client timestamps travel as `local_created_at`/`local_updated_at`; the server orders changes by `update_seq`.
- Who triggers: every write use case, pull-to-refresh (`SyncUseCase`), app start (`MainViewModel`), login/logout, FCM token
  registration.
- Logout: `LogoutUseCase` → API logout → clear user, token → clear tables in one transaction → `sync()` (config only).
- legacy: the loop is copy-pasted in `NoteRepositoryImpl`, `ContactRepositoryImpl`, `DebtRepositoryImpl`; `NoteRepositoryImpl`
  marks rows synced even if the server rejected them over the limit (`// TODO server not sync limit`).
  **[proposal]** one generic `SyncEngine<Entity, RequestDto, ResponseDto>` in `data/sync/` and per-row acknowledgements from
  the server.

## Platform data sources

- Contacts picker: `ContentResolver.query(uri, DISPLAY_NAME)` inside `withContext(Dispatchers.IO)`, `cursor.use { }`.
- Speech: `callbackFlow` over `SpeechRecognizer`, `awaitClose { stopListening(); destroy() }`.
- Firebase Installations id for push routing: `UserRepository.registerInstallationId()`; deleted on logout.
