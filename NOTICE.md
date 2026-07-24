# Third-Party Notices

Copyright ownCloud GmbH - A Kiteworks Company and ownCloud contributors.

This project builds and distributes a binary compiled from
[owncloud/ocis-workflows](https://github.com/owncloud/ocis-workflows), which is licensed
under Apache-2.0 (see `LICENSE`) but depends on the following third-party component not
licensed under Apache-2.0:

- **[github.com/unraid/apprise-go](https://github.com/unraid/apprise-go)** — BSD-2-Clause,
  Copyright (c) 2025, Chris Caron. Used by `backend/pkg/notify` to implement the workflow
  "notify" action, and compiled into the binary this image ships. License text:
  `LICENSES/BSD-2-Clause.txt`.
