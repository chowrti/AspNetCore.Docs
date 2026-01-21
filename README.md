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



WITH grants AS (
  SELECT 
    event_time,
    user_identity.email AS changed_by,
    request_params.securable_full_name AS object,
    request_params.securable_type AS obj_type,
    explode(from_json(request_params.changes, 'array<struct<principal:string, privileges:array<string>, grant_option:boolean, action:string>>')) AS change_detail
  FROM system.access.audit
  WHERE service_name = 'unityCatalog'
    AND action_name = 'updatePermissions'
    -- Add filters here
)
SELECT 
  event_time,
  changed_by,
  object,
  obj_type,
  change_detail.principal          AS grantee_or_revokee,
  change_detail.privileges         AS privileges_changed,
  change_detail.grant_option       AS with_grant_option,
  change_detail.action             AS change_action   -- e.g., ADD, REMOVE (if present in newer logs)
FROM grants
ORDER BY event_time DESC;


import smtplib, ssl
from email.message import EmailMessage

SMTP_HOST = "smtp.office365.com"   # change if needed
SMTP_PORT = 587
SMTP_USER = "sender@company.com"
SMTP_PASS = dbutils.secrets.get("email-secrets", "smtp-password")

msg = EmailMessage()
msg["Subject"] = "Databricks Test Email"
msg["From"] = SMTP_USER
msg["To"] = "your.email@company.com"
msg.set_content("✅ This is a test email sent from a Databricks notebook.")

context = ssl.create_default_context()

with smtplib.SMTP(SMTP_HOST, SMTP_PORT) as server:
    server.starttls(context=context)
    server.login(SMTP_USER, SMTP_PASS)
    server.send_message(msg)

print("Email sent successfully")








