~~~ java
// 从GDMP系统获取权限信息  
int privilegeLevel;  
try {  
  CatalogPrivilegeResponse response =  
      fetchPrivilegeFromGDMP(credential, catalogName, databaseName);  
  privilegeLevel = response.getCatalogPrivilege() != null ? response.getCatalogPrivilege() : 0;  
  LOG.info(  
      "Successfully loaded catalog privilege from GDMP: catalog={}, accessKey={}, privilege={}",  
      catalogName,  
      accessKey,  
      privilegeLevel);  
  // 缓存权限信息到Redis  
  PrivilegedCacheManager.putCatalogPrivilege(  
      accessKey, catalogName, privilegeLevel, PRIVILEGE_CACHE_EXPIRY_SECONDS);  
  // 这里如果是首次失败的情况下会不会有更新，如果有更新的话后续1h内做了设置后，就不会立马生效
  if (response.getDatabasePrivilege() != null) {  
    PrivilegedCacheManager.putDatabasePrivilege(  
        accessKey,  
        catalogName,  
        databaseName,  
        response.getDatabasePrivilege(),  
        PRIVILEGE_CACHE_EXPIRY_SECONDS);  
  }  
} catch (Exception e) {  
  LOG.error(  
      "Failed to load catalog privilege from GDMP: catalog={}, accessKey={}, error={}",  
      catalogName,  
      accessKey,  
      e.getMessage(),  
      e);  
  throw new RuntimeException(e);  
}
~~~
