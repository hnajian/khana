# Performance Considerations

- **Cache preloading**: Load frequently used translations on startup
- **Batch translation**: Translate multiple texts in single API call
- **Auto-pruning**: Prevent cache from growing indefinitely
- **Lazy translation**: Use IntersectionObserver for full-page translation
- **Provider selection**: Use free providers when possible to conserve quota
