# mssql-jdbc `set_nocount` change set

Target repo: `microsoft/mssql-jdbc` (main branch checked on April 11, 2026)

This change set adds a JDBC connection property:

- `set_nocount=on|off`

and enforces it before statement execution so the session state matches the connection setting.

---

## 1) `src/main/java/com/microsoft/sqlserver/jdbc/SQLServerDriver.java`

### A. Add the new property to `SQLServerDriverStringProperty`

Add it near the other string-valued connection properties:

```java
enum SQLServerDriverStringProperty {
    APPLICATION_INTENT("applicationIntent", ApplicationIntent.READ_WRITE.toString()),
    APPLICATION_NAME("applicationName", SQLServerDriver.DEFAULT_APP_NAME),

    PREPARE_METHOD("prepareMethod", PrepareMethod.PREPEXEC.toString()),

    DATABASE_NAME("databaseName", ""),
    FAILOVER_PARTNER("failoverPartner", ""),
    HOSTNAME_IN_CERTIFICATE("hostNameInCertificate", ""),
    INSTANCE_NAME("instanceName", ""),
    JAAS_CONFIG_NAME("jaasConfigurationName", "SQLJDBCDriver"),
    PASSWORD("password", ""),
    RESPONSE_BUFFERING("responseBuffering", "adaptive"),
    SELECT_METHOD("selectMethod", "direct"),

    // NEW
    SET_NOCOUNT("set_nocount", OnOffOption.OFF.toString()),

    SERVER_NAME("serverName", ""),
    SERVER_SPN("serverSpn", ""),
    TRUST_STORE("trustStore", ""),
    TRUST_STORE_PASSWORD("trustStorePassword", ""),
    TRUST_STORE_TYPE("trustStoreType", "JKS"),
    USER("user", ""),
    WORKSTATION_ID("workstationID", Util.WSID_NOT_AVAILABLE);
```

### B. Add a choice array for `ON|OFF`

Add this next to the existing `TRUE_FALSE` constant:

```java
private static final String[] TRUE_FALSE = {"true", "false"};
private static final String[] ON_OFF = {OnOffOption.ON.toString(), OnOffOption.OFF.toString()};
```

### C. Expose the property in `DRIVER_PROPERTIES`

Add this inside the `DRIVER_PROPERTIES` array:

```java
new SQLServerDriverPropertyInfo(
    SQLServerDriverStringProperty.SET_NOCOUNT.toString(),
    SQLServerDriverStringProperty.SET_NOCOUNT.getDefaultValue(),
    false,
    ON_OFF
),
```

This makes the property visible through `Driver.getPropertyInfo(...)` and aligns it with the other supported connection properties.

---

## 2) `src/main/java/com/microsoft/sqlserver/jdbc/SQLServerConnection.java`

### A. Add a stored connection-level field

Add this near other connection settings such as `lastUpdateCount`, `selectMethod`, etc.:

```java
/** session SET NOCOUNT option controlled by connection property */
private OnOffOption setNoCount = OnOffOption.OFF;

final OnOffOption getSetNoCount() {
    return setNoCount;
}
```

### B. Parse and store the property with the other connection properties

In the connection-property parsing section of `connectInternal(...)` / connection initialization, add:

```java
String sPropValue = activeConnectionProperties.getProperty(
        SQLServerDriverStringProperty.SET_NOCOUNT.toString());

if (null == sPropValue) {
    sPropValue = SQLServerDriverStringProperty.SET_NOCOUNT.getDefaultValue();
}

setNoCount = OnOffOption.valueOfString(sPropValue);
activeConnectionProperties.setProperty(
        SQLServerDriverStringProperty.SET_NOCOUNT.toString(),
        setNoCount.toString());
```

Put this in the same block where other connection properties are read from `activeConnectionProperties` and normalized.

### C. Add a helper that applies the setting to the session

Add a connection helper method:

```java
void applySetNoCount() throws SQLServerException {
    connectionCommand("SET NOCOUNT " + setNoCount.toString(), "applySetNoCount");
}
```

This uses the driver's internal connection-level command path instead of creating a nested `Statement`, so it avoids recursion and follows the same pattern used by other session-level connection commands in `SQLServerConnection`.

> Note:
> `connectionCommand(...)` already exists in this class in current `mssql-jdbc` builds, based on current stack traces and source references from the repo/issues.

---

## 3) `src/main/java/com/microsoft/sqlserver/jdbc/SQLServerStatement.java`

### Add the session enforcement call in the common execute path

In `executeStatement(TDSCommand newStmtCmd)`, call `connection.applySetNoCount()` immediately before `executeCommand(newStmtCmd)`.

### Before

```java
try {
    // (Re)execute this Statement with the new command
    executeCommand(newStmtCmd);
} catch (SQLServerException e) {
```

### After

```java
try {
    // Ensure session state matches the JDBC connection property before every execution.
    connection.applySetNoCount();

    // (Re)execute this Statement with the new command
    executeCommand(newStmtCmd);
} catch (SQLServerException e) {
```

This is the best central hook because `SQLServerStatement.executeStatement(...)` is the shared path used by statement execution flows, so it covers the main `execute*` variants without duplicating logic in many public methods.

---

## Consolidated patch-style view

```diff
*** a/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerDriver.java
--- b/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerDriver.java
@@
 enum SQLServerDriverStringProperty {
@@
-    SELECT_METHOD("selectMethod", "direct"),
+    SELECT_METHOD("selectMethod", "direct"),
+    SET_NOCOUNT("set_nocount", OnOffOption.OFF.toString()),
@@
- private static final String[] TRUE_FALSE = {"true", "false"};
+ private static final String[] TRUE_FALSE = {"true", "false"};
+ private static final String[] ON_OFF = {OnOffOption.ON.toString(), OnOffOption.OFF.toString()};
@@
     new SQLServerDriverPropertyInfo(
         SQLServerDriverStringProperty.APPLICATION_NAME.toString(),
         SQLServerDriverStringProperty.APPLICATION_NAME.getDefaultValue(),
         false,
         null
     ),
+    new SQLServerDriverPropertyInfo(
+        SQLServerDriverStringProperty.SET_NOCOUNT.toString(),
+        SQLServerDriverStringProperty.SET_NOCOUNT.getDefaultValue(),
+        false,
+        ON_OFF
+    ),

*** a/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerConnection.java
--- b/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerConnection.java
@@
     /** last update count flag */
     private boolean lastUpdateCount;
@@
     final boolean useLastUpdateCount() {
         return lastUpdateCount;
     }
+
+    /** session SET NOCOUNT option controlled by connection property */
+    private OnOffOption setNoCount = OnOffOption.OFF;
+
+    final OnOffOption getSetNoCount() {
+        return setNoCount;
+    }
@@
+    String sPropValue = activeConnectionProperties.getProperty(
+            SQLServerDriverStringProperty.SET_NOCOUNT.toString());
+    if (null == sPropValue) {
+        sPropValue = SQLServerDriverStringProperty.SET_NOCOUNT.getDefaultValue();
+    }
+    setNoCount = OnOffOption.valueOfString(sPropValue);
+    activeConnectionProperties.setProperty(
+            SQLServerDriverStringProperty.SET_NOCOUNT.toString(),
+            setNoCount.toString());
+
@@
+    void applySetNoCount() throws SQLServerException {
+        connectionCommand("SET NOCOUNT " + setNoCount.toString(), "applySetNoCount");
+    }

*** a/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerStatement.java
--- b/src/main/java/com/microsoft/sqlserver/jdbc/SQLServerStatement.java
@@
         try {
+            connection.applySetNoCount();
             executeCommand(newStmtCmd);
         } catch (SQLServerException e) {
```

---

## Behavior

Example JDBC URL:

```text
jdbc:sqlserver://host:1433;databaseName=demo;set_nocount=on
```

Result:

- The property is parsed from the JDBC URL and stored on the connection.
- Before each statement execution through the shared statement path, the driver sends:
  - `SET NOCOUNT ON` when `set_nocount=on`
  - `SET NOCOUNT OFF` when `set_nocount=off`

---

## Important implementation note

This approach is correct for the requirement you gave, but it does add one extra round trip before each statement execution because it re-applies the session setting every time.

If you want a lower-overhead version, the next iteration would be:

- track the last applied NOCOUNT state on the connection
- only call `SET NOCOUNT ...` when the driver believes the session state is out of sync

That is faster, but it is less strict if application SQL manually changes `SET NOCOUNT` between executes.

---

## Suggested follow-up test cases

1. `jdbc:sqlserver://...;set_nocount=on`
   - verify session honors NOCOUNT ON

2. `jdbc:sqlserver://...;set_nocount=off`
   - verify update counts behave normally

3. invalid value:
   - `set_nocount=true`
   - `set_nocount=1`
   - should raise invalid connection setting

4. case-insensitive values:
   - `set_nocount=ON`
   - `set_nocount=off`

5. prepared statement and callable statement execution
   - verify shared execute path applies NOCOUNT setting there too


docker pull mcr.microsoft.com/mssql/server:2022-latest


docker run -e "ACCEPT_EULA=Y" `
-e "MSSQL_SA_PASSWORD=YourStrong(!)Password" `
-p 1433:1433 `
--name sqlserver `
-d mcr.microsoft.com/mssql/server:2022-latest