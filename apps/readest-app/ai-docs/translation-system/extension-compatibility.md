# Extension Compatibility

**Support for Translation Extensions** (v0.9.36, commit #901):

The system supports integration with browser extensions like LingKuma and Immersive Translate:

- **Iframe Accessibility**: Minimal sandbox restrictions to allow extension injection
- **Message Passing**: Iframe events forwarded to parent for extension communication
- **Foliate-js Updates**: Submodule updated with more accessible iframe attributes

This allows users to use their preferred translation extension alongside Readest's built-in translation system.
