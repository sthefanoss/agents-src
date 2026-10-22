---
name: debloat-mac
description: Disables macOS AI, Siri, and media-indexing LaunchAgents and terminates their processes, or re-enables the agents. Use on a direction to debloat macOS, to stop macOS background AI and indexing daemons, or to bring them back. Debloating or restoring booted iOS Simulators belongs to debloat-simulators.
---

# Debloat macOS

Run this script in `zsh`. To undo an earlier run, change its first line to `MODE=enable`. Keep `VisionKit` (`vkd`) out of the target lists so on-demand Live Text OCR continues to work:

```zsh
MODE=disable

set -u
case $MODE in
  enable|disable) ;;
  *) echo "MODE must be enable or disable, got: ${MODE}"; exit 1 ;;
esac

DOMAIN="gui/$(id -u)"
AGENT_DIRS=(
  /System/Library/LaunchAgents
  /Library/Apple/System/Library/LaunchAgents
  /System/Volumes/Preboot/Cryptexes/App/System/Library/LaunchAgents
)

MACOS_AI_SERVICES=(
  "com.apple.campo"
  "com.apple.generativeexperiencesd"
  "com.apple.GenerativeFunctions.agentstored"
  "com.apple.intelligenceplatformd"
  "com.apple.intelligenceflowd"
  "com.apple.intelligencecontextd"
  "com.apple.intelligencetasksd"
  "com.apple.siriinferenced"
  "com.apple.siriknowledged"
  "com.apple.siriappintentsd"
  "com.apple.siriactionsd"
  "com.apple.siri.context.service"
  "com.apple.assistantd"
  "com.apple.assistant_service"
  "com.apple.assistant_cdmd"
  "com.apple.SiriTTSTrainingAgent"
  "com.apple.sirittsd"
  "com.apple.suggestd"
  "com.apple.mediaanalysisd"
  "com.apple.photoanalysisd"
  "com.apple.knowledge-agent"
)

for service in "${MACOS_AI_SERVICES[@]}"; do
  launchctl "$MODE" "${DOMAIN}/${service}"
  [[ $MODE == disable ]] && launchctl bootout "${DOMAIN}/${service}" 2>/dev/null
  plists=( ${^AGENT_DIRS}/${service}.plist(N) )
  if ! launchctl print-disabled "$DOMAIN" | grep -qF "\"${service}\" => ${MODE}d"; then
    echo "  [FAILED] ${service}"
  elif (( ${#plists} == 0 )); then
    echo "  [ABSENT] ${service}"
  else
    echo "  [${(U)MODE}D] ${service}"
  fi
done

PROCESSES_TO_KILL=(
  "Siri AI"
  "generativeexperiencesd"
  "GenerativeExperiencesSafetyInferenceProvider"
  "intelligenceplatformd"
  "IntelligencePlatformComputeService"
  "intelligenceflowd"
  "intelligencecontextd"
  "intelligencetasksd"
  "siriinferenced"
  "siriknowledged"
  "siriappintentsd"
  "siriactionsd"
  "SiriSuggestionsBookkeepingService"
  "SiriSetupSettingsIntents"
  "SiriAUSP"
  "assistantd"
  "assistant_service"
  "assistant_cdmd"
  "suggestd"
  "mediaanalysisd"
  "photoanalysisd"
  "knowledge-agent"
)

if [[ $MODE == disable ]]; then
  for proc in "${PROCESSES_TO_KILL[@]}"; do
    killall -9 "$proc" 2>/dev/null && echo "  [KILLED] ${proc}"
  done
  sleep 5
  for proc in "${PROCESSES_TO_KILL[@]}"; do
    killall -s "$proc" >/dev/null 2>&1 && echo "  [RESPAWNED] ${proc}"
  done
fi

vm_stat | head -n 10
sysctl vm.swapusage
```

`[ABSENT]` marks a label with no plist on this macOS. The script still writes its override, so a later macOS that ships the agent finds it already disabled, and an undo clears what an earlier run wrote.

With SIP enabled, `launchctl bootout` fails, so an agent that was already loaded stays loaded and launchd relaunches it after the kill. That is what `[RESPAWNED]` reports. launchctl(1) says a disabled service cannot be loaded until it is enabled again, and that this persists across boots. Tell the user that a respawned process run by one of these labels should stay off after the next logout or restart, and that only a run after one confirms it. A respawned XPC service such as `SiriSuggestionsBookkeepingService` belongs to no label here, so promise nothing for it.

launchctl has no command that removes an override, so an undo leaves every label explicitly enabled. That also turns on `com.apple.campo` and `com.apple.intelligenceflowd` where Apple's feature flags would keep them off, since both plists carry a feature-flag `Disabled` key that the override replaces. Say so when reporting an undo.

Report the count of each status, every `[FAILED]`, `[KILLED]`, and `[RESPAWNED]` entry by name, and the `vm.swapusage` line in the reply.
