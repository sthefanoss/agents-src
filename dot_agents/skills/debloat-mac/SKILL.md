---
name: debloat-mac
description: Disables and terminates macOS AI, Siri, and media-indexing LaunchAgents. Use on a direction to debloat macOS or stop macOS background AI and indexing daemons. Debloating booted iOS Simulators belongs to debloat-simulators.
---

# Debloat macOS

Run this script in `zsh`. Keep `VisionKit` (`vkd`) out of the target lists so on-demand Live Text OCR continues to work:

```zsh
set -u

UID_VAL=$(id -u)

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
)

for service in "${MACOS_AI_SERVICES[@]}"; do
  launchctl disable "gui/${UID_VAL}/${service}" 2>/dev/null || true
  launchctl bootout "gui/${UID_VAL}/${service}" 2>/dev/null || true
  echo "  [OK] Disabled: ${service}"
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
)

for proc in "${PROCESSES_TO_KILL[@]}"; do
  if killall -9 "$proc" 2>/dev/null; then
    echo "  [KILLED] Terminated: $proc"
  fi
done

vm_stat | head -n 10
sysctl vm.swapusage
```

Report the disabled service count, the terminated processes, and the `vm.swapusage` line in the reply.
