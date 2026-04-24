##### 1.从环境变量中获取配置
~~~properties
CATALOG_CATALOG__IMPL=org.apache.iceberg.jdbc.JdbcCatalog -> catalog-impl=org.apache.iceberg.jdbc.JdbcCatalog  
CATALOG_URI=jdbc:sqlite:memory: -> uri=jdbc:sqlite:memory:  
CATALOG_WAREHOUSE=test_warehouse -> warehouse=test_warehouse     
CATALOG_IO__IMPL=org.apache.iceberg.aws.s3.S3FileIO -> io-impl=org.apache.iceberg.aws.s3.S3FileIO  
CATALOG_JDBC_USER=ice_user -> jdbc.user=ice_user
~~~
~~~java
static Map<String, String> environmentCatalogConfig() {  
  return System.getenv().entrySet().stream()  
      .filter(e -> e.getKey().startsWith(CATALOG_ENV_PREFIX))  
      .collect(  
          Collectors.toMap(  
              e ->  
                  e.getKey()  
                      .replaceFirst(CATALOG_ENV_PREFIX, "")  
                      .replaceAll("__", "-")  
                      .replaceAll("_", ".")  
                      .toLowerCase(Locale.ROOT),  
              Map.Entry::getValue,  
              (m1, m2) -> {  
                throw new IllegalArgumentException("Duplicate key: " + m1);  
              },  
              HashMap::new));  
}
~~~
##### 2.主要的catalog抽象
~~~java
// 这里定义了全部的operation，在各个impl中具体实现
public interface SessionCatalog {

final class SessionContext {  
  private final String sessionId;  
  private final String identity;  
  private final Map<String, String> credentials;  
  private final Map<String, String> properties;  
  private final Object wrappedIdentity;
  // ... 略
}

/**  
 * Return all the identifiers under this namespace. * * @param context session context  
 * @param namespace a namespace  
 * @return a list of identifiers for tables  
 * @throws NoSuchNamespaceException if the namespace does not exist  
 */List<TableIdentifier> listTables(SessionContext context, Namespace namespace);  
  
/**  
 * Create a builder to create a table or start a create/replace transaction. * * @param context session context  
 * @param ident a table identifier  
 * @param schema a schema  
 * @return the builder to create a table or start a create/replace transaction  
 */Catalog.TableBuilder buildTable(SessionContext context, TableIdentifier ident, Schema schema);  
  
/**  
 * Register a table if it does not exist. * * @param context session context  
 * @param ident a table identifier  
 * @param metadataFileLocation the location of a metadata file  
 * @return a Table instance  
 * @throws AlreadyExistsException if the table already exists in the catalog.  
 */Table registerTable(SessionContext context, TableIdentifier ident, String metadataFileLocation);  
  
/**  
 * Check whether table exists. * * @param context session context  
 * @param ident a table identifier  
 * @return true if the table exists, false otherwise  
 */default boolean tableExists(SessionContext context, TableIdentifier ident) {  
  try {  
    loadTable(context, ident);  
    return true;  
  } catch (NoSuchTableException e) {  
    return false;  
  }  
}  
  
/**  
 * Load a table. * * @param context session context  
 * @param ident a table identifier  
 * @return instance of {@link Table} implementation referred by {@code tableIdentifier}  
 * @throws NoSuchTableException if the table does not exist  
 */Table loadTable(SessionContext context, TableIdentifier ident);  
  
/**  
 * Drop a table, without requesting that files are immediately deleted. * * <p>Data and metadata files should be deleted according to the catalog's policy. * * @param context session context  
 * @param ident a table identifier  
 * @return true if the table was dropped, false if the table did not exist  
 */boolean dropTable(SessionContext context, TableIdentifier ident);  
  
/**  
 * Drop a table and request that files are immediately deleted. * * @param context session context  
 * @param ident a table identifier  
 * @return true if the table was dropped and purged, false if the table did not exist  
 * @throws UnsupportedOperationException if immediate delete is not supported  
 */boolean purgeTable(SessionContext context, TableIdentifier ident);  
  
/**  
 * Rename a table. * * @param context session context  
 * @param from identifier of the table to rename  
 * @param to new table name  
 * @throws NoSuchTableException if the from table does not exist  
 * @throws AlreadyExistsException if the to table already exists  
 */void renameTable(SessionContext context, TableIdentifier from, TableIdentifier to);  
  
/**  
 * Invalidate cached table metadata from current catalog. * * <p>If the table is already loaded or cached, drop cached data. If the table does not exist or * is not cached, do nothing. * * @param context session context  
 * @param ident a table identifier  
 */void invalidateTable(SessionContext context, TableIdentifier ident);  
  
/**  
 * Create a namespace in the catalog. * * @param context session context  
 * @param namespace a {@link Namespace namespace}  
 * @throws AlreadyExistsException If the namespace already exists  
 * @throws UnsupportedOperationException If create is not a supported operation  
 */default void createNamespace(SessionContext context, Namespace namespace) {  
  createNamespace(context, namespace, ImmutableMap.of());  
}  
  
/**  
 * Create a namespace in the catalog. * * @param context session context  
 * @param namespace a {@link Namespace namespace}  
 * @param metadata a string Map of properties for the given namespace  
 * @throws AlreadyExistsException If the namespace already exists  
 * @throws UnsupportedOperationException If create is not a supported operation  
 */void createNamespace(SessionContext context, Namespace namespace, Map<String, String> metadata);  
  
/**  
 * List top-level namespaces from the catalog. * * <p>If an object such as a table, view, or function exists, its parent namespaces must also * exist and must be returned by this discovery method. For example, if table a.b.t exists, this * method must return ["a"] in the result array. * * @param context session context  
 * @return a List of namespace {@link Namespace} names  
 */default List<Namespace> listNamespaces(SessionContext context) {  
  return listNamespaces(context, Namespace.empty());  
}  
  
/**  
 * List child namespaces from the namespace. * * <p>For two existing tables named 'a.b.c.table' and 'a.b.d.table', this method returns: * * <ul> *   <li>Given: {@code Namespace.empty()}  
 *   <li>Returns: {@code Namespace.of("a")}  
 * </ul> * * <ul> *   <li>Given: {@code Namespace.of("a")}  
 *   <li>Returns: {@code Namespace.of("a", "b")}  
 * </ul> * * <ul> *   <li>Given: {@code Namespace.of("a", "b")}  
 *   <li>Returns: {@code Namespace.of("a", "b", "c")} and {@code Namespace.of("a", "b", "d")}  
 * </ul> * * <ul> *   <li>Given: {@code Namespace.of("a", "b", "c")}  
 *   <li>Returns: empty list, because there are no child namespaces * </ul> * * @param context session context  
 * @param namespace a {@link Namespace namespace}  
 * @return a List of child {@link Namespace} names from the given namespace  
 * @throws NoSuchNamespaceException If the namespace does not exist (optional)  
 */List<Namespace> listNamespaces(SessionContext context, Namespace namespace);  
  
/**  
 * Load metadata properties for a namespace. * * @param context session context  
 * @param namespace a {@link Namespace namespace}  
 * @return a string map of properties for the given namespace  
 * @throws NoSuchNamespaceException If the namespace does not exist (optional)  
 */Map<String, String> loadNamespaceMetadata(SessionContext context, Namespace namespace);  
  
/**  
 * Drop a namespace. If the namespace exists and was dropped, this will return true. * * @param context session context  
 * @param namespace a {@link Namespace namespace}  
 * @return true if the namespace was dropped, false otherwise.  
 * @throws NamespaceNotEmptyException If the namespace is not empty  
 */boolean dropNamespace(SessionContext context, Namespace namespace);  
  
/**  
 * Set a collection of properties on a namespace in the catalog. * * <p>Properties that are not in the given map are not modified or removed by this method. * * @param context session context  
 * @param namespace a {@link Namespace namespace}  
 * @param updates properties to set for the namespace  
 * @param removals properties to remove from the namespace  
 * @throws NoSuchNamespaceException If the namespace does not exist (optional)  
 * @throws UnsupportedOperationException If namespace properties are not supported  
 */boolean updateNamespaceMetadata(  
    SessionContext context,  
    Namespace namespace,  
    Map<String, String> updates,  
    Set<String> removals);  
  
/**  
 * Checks whether the Namespace exists. * * @param context session context  
 * @param namespace a {@link Namespace namespace}  
 * @return true if the Namespace exists, false otherwise  
 */default boolean namespaceExists(SessionContext context, Namespace namespace) {  
  try {  
    loadNamespaceMetadata(context, namespace);  
    return true;  
  } catch (NoSuchNamespaceException e) {  
    return false;  
  }  
}
~~~
- 在rest catalog里面的主要实现就是RESTSessionCatalog
![[RESTSessionCatalog.png]]
- *实际逻辑都在CatalogHandlers.java中定义，重要成员是一个catalog的句柄，具体落入到jdbc or or*
- *具体操作数据库的代码咋JdbcUtil.java中定义*


