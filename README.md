# goodreads

[![Go Reference](https://pkg.go.dev/badge/github.com/icco/goodreads.svg)](https://pkg.go.dev/github.com/icco/goodreads)
[![Test Go](https://github.com/icco/goodreads/actions/workflows/test.yml/badge.svg)](https://github.com/icco/goodreads/actions/workflows/test.yml)

A Go client for reading a user's public [Goodreads](https://www.goodreads.com) bookshelves.

**Goodreads retired its official API in December 2020 and has issued no new keys since.** Every Go client that wraps that API is dead code today. The per-shelf RSS feed is the only route still open, and it is a good one: no key, no auth, no registration for a public profile, and it carries book id, title, author, ISBN, community average rating, the user's own stars, page count, publication year, shelf names, cover art, and read/added dates.

```
go get github.com/icco/goodreads
```

## Usage

```go
c := goodreads.NewClient()

// The id is the number in your profile URL:
// goodreads.com/user/show/12680-nat  ->  "12680"
books, err := c.Shelf(ctx, "12680", goodreads.ShelfRead)
if errors.Is(err, goodreads.ErrTruncated) {
  // books is still valid, just incomplete. Raise c.MaxPages.
  log.Printf("shelf capped: %v", err)
} else if err != nil {
  return err
}

for _, b := range books {
  fmt.Printf("%s by %s — you: %d/5, everyone: %.2f/5\n",
    b.Title, b.Author, b.UserRating, b.AverageRating)
}
```

`ShelfRead` ("read") and `ShelfToRead` ("to-read") are provided as constants, but any shelf name works, including custom ones.

## Notes

- **Only public profiles.** There is no auth, so a private profile 404s. `Shelf` returns an error for that rather than an empty slice — an empty shelf and an unreadable one are very different things, and conflating them silently wipes whatever you were building.
- **The user id is not the author id.** `goodreads.com/user/show/<id>` and `goodreads.com/author/show/<id>` are different namespaces. Passing an author id returns some unrelated person's shelves *without an error*, so this mistake fails plausibly rather than loudly.
- **`ErrTruncated` comes back with partial results.** Paging stops at `MaxPages` (default 40, or 4,000 books). If a shelf is bigger, you get the books read so far *and* the error, so a capped pool is never mistaken for a complete one. Check it with `errors.Is` and raise `MaxPages` if you need to.
- **Shelf names are thin in practice.** `user_shelves` holds the user's own shelf names, not genres, and most people barely use them — measured on a real 1,131-book profile: 4 "fiction", 3 "digital-owned", 1 "top-ten". Star ratings are far richer. If you want a taste signal, use `UserRating`, not `Shelves`. Bookkeeping names ("read", "to-read", "currently-reading") are stripped.
- **`AverageRating` is on Goodreads' native 0–5 scale**, not rescaled.
- **`UserRating` of 0 means unrated**, not "rated zero" — Goodreads has no zero-star rating.
- **A feed that stops advancing terminates the walk.** If a page repeats content already seen, paging stops rather than looping to `MaxPages`.
- **Read only.** RSS is a read interface; there is no way to write through it.
- No third-party dependencies.

## License

MIT
