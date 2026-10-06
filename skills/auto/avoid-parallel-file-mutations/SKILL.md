---
name: avoid-parallel-file-mutations
description: When making changes to multiple files, ensure that no two modifications target the same file simultaneously.
---
- Review all planned file modifications before executing them.
- Ensure that each file is modified in a separate operation.
- If multiple changes are needed for the same file, batch them into a single operation.
- Use version control to track changes and avoid conflicts.
- Test each modification individually to confirm it works as expected before proceeding to the next.
- Document the changes made to each file for clarity and future reference.