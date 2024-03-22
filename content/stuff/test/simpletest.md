---
title: Simpletest has images and code

themes:
  - development
  - annoyance
  - testing
---

## A header

Here's some text to start us off.

```go { caption="Look, it's some Go code! How about that?" }
func main() {
  logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
  flag.Parse()
  if *address == "" {
    logger.Error("Can't bind an empty address")
    os.Exit(1)
  }

  http.Handle("/", echo.New(logger))
  logger.Info("Server starting", "address", *address)

  err := http.ListenAndServe(*address, nil)
  logger.Info("Server has exited", "error", err)
}
```
