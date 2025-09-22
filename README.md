-- разово: CREATE EXTENSION IF NOT EXISTS pgcrypto;

INSERT INTO public."Process"
("Id","ProcessCode","ProcessName","StartBlockId","Created","Updated","CreateUserId","LastUserId")
VALUES (
  gen_random_uuid(),
  'ServiceRequest',
  'Техническая поддержка',
  NULL,
  NOW(),
  NOW(),
  'System',
  'System'
);
