-- разово: CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

INSERT INTO public."Process"
("Id","ProcessCode","ProcessName","StartBlockId","Created","Updated","CreateUserId","LastUserId")
VALUES (
  uuid_generate_v4(),
  'ServiceRequest',
  'Техническая поддержка',
  NULL,
  NOW(),
  NOW(),
  'System',
  'System'
);
