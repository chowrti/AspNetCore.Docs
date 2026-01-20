SELECT 
  grantee AS user_email,
  privilege_type,
  CASE 
    WHEN table_catalog IS NOT NULL THEN CONCAT(table_catalog, '.', table_schema, '.', table_name)
    WHEN catalog_name IS NOT NULL AND schema_name IS NOT NULL THEN CONCAT(catalog_name, '.', schema_name)
    WHEN catalog_name IS NOT NULL THEN catalog_name
  END AS object_path,
  grantor,
  is_grantable
FROM (
  SELECT grantee, privilege_type, catalog_name, NULL AS schema_name, NULL AS table_name, grantor, is_grantable FROM system.information_schema.catalog_privileges
  UNION ALL
  SELECT grantee, privilege_type, catalog_name, schema_name, NULL, grantor, is_grantable FROM system.information_schema.schema_privileges
  UNION ALL
  SELECT grantee, privilege_type, table_catalog, table_schema, table_name, grantor, is_grantable FROM system.information_schema.table_privileges
) t
WHERE grantee LIKE '%@%' 
  AND grantee NOT REGEXP '^[0-9a-f-]{36}$'
ORDER BY grantee, object_path;
