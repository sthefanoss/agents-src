---
name: debloat-simulators
description: Disables and terminates background AI, Siri, Spotlight, and analysis daemons in booted iOS Simulators. Use on a direction to debloat or optimize running iOS Simulators. Debloating the host macOS system belongs to debloat-mac.
---

# Debloat iOS Simulators

Run this script in `zsh`:

```zsh
set -u

UID_VAL=$(id -u)

SIM_BLOAT_DAEMONS=(
  "com.apple.siriactionsd"
  "com.apple.siriinferenced"
  "com.apple.siriknowledged"
  "com.apple.siriappintentsd"
  "com.apple.assistantd"
  "com.apple.assistant_service"
  "com.apple.assistant_cdmd"
  "com.apple.intelligenceplatformd"
  "com.apple.intelligenceflowd"
  "com.apple.intelligencecontextd"
  "com.apple.generativeexperiencesd"
  "com.apple.suggestd"
  "com.apple.duetexpertd"
  "com.apple.biomesyncd"
  "com.apple.proactiveeventtrackerd"
  "com.apple.ContextStoreAgent"
  "com.apple.parsecd"
  "com.apple.spotlightknowledged"
  "com.apple.spotlightknowledged.updater"
  "com.apple.corespotlightd"
  "com.apple.searchd"
  "com.apple.mediaanalysisd"
  "com.apple.photoanalysisd"
)

SIM_BLOAT_PROCS=(
  "siriactionsd"
  "siriinferenced"
  "siriknowledged"
  "siriappintentsd"
  "assistantd"
  "assistant_cdmd"
  "intelligenceplatformd"
  "intelligenceflowd"
  "intelligencecontextd"
  "generativeexperiencesd"
  "suggestd"
  "duetexpertd"
  "biomesyncd"
  "proactiveeventtrackerd"
  "ContextStoreAgent"
  "parsecd"
  "spotlightknowledged"
  "spotlightknowledged.updater"
  "corespotlightd"
  "searchd"
  "mediaanalysisd"
  "photoanalysisd"
  "HistoricalAnalyzerService"
  "DayStreamProcessorService"
)

BOOTED_UDIDS=($(xcrun simctl list devices booted 2>/dev/null | grep -E "\([0-9A-F-]{36}\) \(Booted\)" | sed -E 's/.*\(([0-9A-F-]{36})\) \(Booted\).*/\1/'))

if [ ${#BOOTED_UDIDS[@]} -eq 0 ]; then
  echo "[!] No Booted iOS Simulator found."
else
  for udid in "${BOOTED_UDIDS[@]}"; do
    SIM_NAME=$(xcrun simctl list devices booted | grep "$udid" | sed -E 's/^[[:space:]]*([^ (]+.*)[[:space:]]+\([0-9A-F-]{36}\).*/\1/')
    echo ">>> Optimizing Simulator: ${SIM_NAME} [${udid}]"

    xcrun simctl spawn "$udid" defaults write com.apple.assistant.support "Assistant Enabled" -bool false 2>/dev/null || true
    xcrun simctl spawn "$udid" defaults write com.apple.Siri SiriEnabled -bool false 2>/dev/null || true
    xcrun simctl spawn "$udid" defaults write com.apple.Spotlight SuggestionsEnabled -bool false 2>/dev/null || true
    xcrun simctl spawn "$udid" defaults write com.apple.Accessibility ReduceMotionEnabled -int 1 2>/dev/null || true
    xcrun simctl spawn "$udid" defaults write com.apple.Accessibility EnhancedBackgroundContrastEnabled -int 1 2>/dev/null || true

    for daemon in "${SIM_BLOAT_DAEMONS[@]}"; do
      xcrun simctl spawn "$udid" launchctl disable "system/${daemon}" 2>/dev/null || true
      xcrun simctl spawn "$udid" launchctl disable "user/${UID_VAL}/${daemon}" 2>/dev/null || true
      xcrun simctl spawn "$udid" launchctl disable "gui/${UID_VAL}/${daemon}" 2>/dev/null || true
      xcrun simctl spawn "$udid" launchctl bootout "system/${daemon}" 2>/dev/null || true
      xcrun simctl spawn "$udid" launchctl bootout "gui/${UID_VAL}/${daemon}" 2>/dev/null || true
    done
    echo "  [OK] ${#SIM_BLOAT_DAEMONS[@]} daemons disabled in launchd_sim."

    KILLED_COUNT=0
    for proc in "${SIM_BLOAT_PROCS[@]}"; do
      if xcrun simctl spawn "$udid" killall -9 "$proc" 2>/dev/null; then
        KILLED_COUNT=$((KILLED_COUNT + 1))
      fi
    done
    echo "  [OK] ${KILLED_COUNT} processes terminated inside simulator."
  done
fi

ps -Ao rss,comm | awk '
  /CoreSimulator|iOS_/ { count++; rss += $1 }
  END {
    printf "CoreSimulator processes: %d | Total RSS: %.1f MB\n", count+0, rss/1024
  }
'
```

Report each optimized simulator UDID, the terminated process count, and the `CoreSimulator` total RSS in the reply.
