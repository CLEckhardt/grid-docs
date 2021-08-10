# Updating Grid Database Schema

<!--
  Copyright (c) 2018-2021 Cargill Incorporated
  Licensed under Creative Commons Attribution 4.0 International License
  https://creativecommons.org/licenses/by/4.0/
-->

Occasionally, the Grid database schema may need to be updated with new or
modified tables. These changes are made by adding new migrations files for
each Grid-supported database backend.

New migrations can be generated by navigating to the
`sdk/src/migrations/diesel/<backend>` directory for the backend the migration
is for and running the command:

```
$ diesel migration generate $NEW_MIGRATION_NAME
```

Migrations consist of two files, `up.sql` and `down.sql`. The `up.sql` file
contains the database commands to run the required updates while the `down.sql`
file reverses those same changes.

When adding new migrations it is a good idea to test that the migrations run
successfully. To run the migrations, start the Grid database for the appropriate
backend and use the command:

```
$ grid database migrate
```

## Other Considerations

New migrations should take care when updating tables not to delete any data
unintentionally. For example, adding a column to a table by dropping it and
adding the updated table would update the schema but could result in data loss
for a user updating their own system. Instead, `ALTER TABLE` commands should be
used where possible.