# Decision records

Each file records one architecture or project decision: the situation, what we chose and what follows from it. Git history shows *what* changed. These files explain *why*.

## How to add one

1. Copy [`0000-template.md`](0000-template.md) to the next number, for example `0002-short-title.md`.
2. Fill it in. A few short paragraphs is enough.
3. Add it to the index below in the same pull request as the change it describes.

Don't edit or delete an accepted record. If a decision changes, write a new record that supersedes the old one, then set the old one's status to `Superseded by 000N`.

## Index

| # | Decision | Status | Date |
| --- | --- | --- | --- |
| [0001](0001-maven-over-gradle.md) | Use Maven, not Gradle, for the Java build | Accepted | 2026-11-09 |
