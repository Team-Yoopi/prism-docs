# Prism Video Documentation

Documentation for the Prism Video Player ecosystem.

**Live site:** https://prism.teamyoopi.dev/

## Contents

- [Home](index.md) - Overview and quick start
- [Testing Guide](testing.md) - Stream testing instructions
- [Plugin API](api.md) - Plugin development reference
- [Architecture](architecture.md) - System design overview
- [Building](building.md) - Build instructions

## Local Development

```bash
# Install Jekyll
gem install bundler jekyll

# Serve locally
bundle exec jekyll serve

# Open http://localhost:4000/prism-docs/
```

## Related Repositories

| Repository | Description |
|------------|-------------|
| [prism-video](https://github.com/Team-Yoopi/prism-video) | Core player (private) |
| [prism-ffmpeg-plugin](https://github.com/Team-Yoopi/prism-ffmpeg-plugin) | FFmpeg decoder |
| [prism-ytdlp-plugin](https://github.com/Team-Yoopi/prism-ytdlp-plugin) | URL resolver |
| [prism-unity-plugin](https://github.com/Team-Yoopi/prism-unity-plugin) | Unity integration |
