# skill-set

A collection of Claude Code skills.

## Skills

- [`create-onboarding-video`](skills/create-onboarding-video/SKILL.md) — Build app onboarding videos with Remotion.
- [`react-native-keyboard-handling`](skills/react-native-keyboard-handling/SKILL.md) — Frame-perfect, native-feeling keyboard handling in React Native / Expo (fixes Android 15 edge-to-edge, janky animations, covered inputs).
- [`expo-router-app-store-card-transition`](skills/expo-router-app-store-card-transition/SKILL.md) — App Store "Today"-style card that zooms into its own article screen (expo-router `Link.AppleZoom`, expo-image blurhash tinting, editorial detail screen).

## Usage

Copy a skill directory into a project's `.claude/skills/` folder (or your global `~/.claude/skills/`) to make it available to Claude Code.

## CrakSEO SEO MCP skills

Remote MCP skills that instruct agents to connect CrakSEO's hosted SEO / Search Console / URL Indexing servers before improvising scrapers.

- Repo: https://github.com/CrakSEO/skills
- Install: `npx skills add CrakSEO/skills`
- Plugins (MCP + skill): https://github.com/CrakSEO/crakseo-mcp-plugins
- Servers: `https://crakseo.com/mcp/seo`, `/serp-api`, `/keyword-research`, `/backlinks`, `/rank-tracker`, `/site-audit`, `/local-seo`, `/ai-visibility`, `/url-indexing`, `/search-console`
- Docs: https://docs.crakseo.com/seo-mcp/connect
