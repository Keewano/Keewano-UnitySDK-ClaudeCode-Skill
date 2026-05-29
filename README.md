# Keewano Unity SDK — Claude Code Skill

A [Claude Code](https://claude.com/claude-code) skill that turns Claude into a proactive integration partner for the [Keewano Unity SDK](https://github.com/Keewano/Keewano-UnitySDK). Once installed, Claude surveys your Unity project for instrumentation gaps, proposes concrete event reports at specific file/line locations, flags anti-patterns in existing usage, and helps you produce an event stream that tells the player's story.

Optional — install only if you use Claude Code. The Keewano Unity SDK itself does not require this.

## The SDK

**Download:** Keewano Unity SDK as a `.unitypackage` from the [GitHub releases page](https://github.com/Keewano/Keewano-UnitySDK/releases).

**Documentation:** <https://keewano.github.io/Keewano-UnitySDK/>

## Install

Copy the `keewano-unity-sdk/` folder into Claude Code's skills directory.

**Project-scoped** (versioned with your game; available to everyone on the team):

```
<your-unity-project>/.claude/skills/keewano-unity-sdk/SKILL.md
```

**User-scoped** (available across all your Claude Code sessions):

```
~/.claude/skills/keewano-unity-sdk/SKILL.md
```

**Restart Claude Code** (or start a new session) so it loads the skill. After that, Claude auto-activates it when you work on Keewano-related code.

## Verify

Ask Claude: *"What skills do you have available?"* — `keewano-unity-sdk` should be in the list.

## Updating

When you upgrade the Keewano Unity SDK, pull the latest version of this repo and re-copy the skill folder.

## Feedback

Pattern Claude handles badly, or a case it should know about? Email <support@keewano.com> with the prompt and Claude's output — we'll fold it into the next release.

## License

Apache 2.0 — see [LICENSE.md](LICENSE.md).
