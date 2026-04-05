# How the Voice System Works

> This document explains the complete voice pipeline in the BioT Speech IoT app, from the user speaking, through speech recognition, intent matching, action execution, and spoken response. No prior knowledge of voice systems required.

---

## Table of Contents

1. [Overview: The Big Picture](#1-overview--the-big-picture)
2. [The Voice Package Structure](#2-the-voice-package-structure)
3. [Speech-to-Text: VoiceInputManager](#3-speech-to-text--voiceinputmanager)
4. [The Command Dictionary: VoiceCommandDictionary](#4-the-command-dictionary--voicecommanddictionary)
5. [Intent Matching: VoiceCommandResolver](#5-intent-matching--voicecommandresolver)
6. [All Available Commands: VoiceCommand enum](#6-all-available-commands--voicecommand-enum)
7. [Executing Actions: VoiceCommandExecutor](#7-executing-actions--voicecommandexecutor)
8. [Text-to-Speech: TtsManager](#8-text-to-speech--ttsmanager)
9. [The Three Interfaces](#9-the-three-interfaces)
10. [Emulator vs Real Device](#10-emulator-vs-real-device)
11. [The Complete Flow: Step by Step](#11-the-complete-flow--step-by-step)
12. [Adding New Commands](#12-adding-new-commands)

---

## 1. Overview: The Big Picture

The voice system turns spoken words into app actions. It is entirely self-contained inside the `com.fhdw.biot.speech.iot.voice` package, no LLM or external API is involved at this stage. Everything runs on-device using Android's built-in speech recognition and text-to-speech engines.

The full pipeline looks like this:

```
User speaks
    ↓
SpeechRecognizer (Android built-in)
    ↓ transcript as plain text
VoiceCommandResolver  ←  VoiceCommandDictionary (keyword rules)
    ↓ matched VoiceCommand enum value
VoiceCommandExecutor
    ↓ calls the right app action
Navigate screen / Filter chart / Change MQTT mode / etc.
    ↓
TtsManager speaks the result back to the user
```

> **Note:** The key design decision is that the matching logic (`VoiceCommandResolver`) has no Android dependencies, it just takes a string and returns a `VoiceCommand`. This means it can be unit tested without an Android emulator running.

---

## 2. The Voice Package Structure

All voice code lives in one package:

```
com.fhdw.biot.speech.iot.voice/
├── VoiceCommand.java           — enum of every possible command intent
├── VoiceCommandDictionary.java — maps keywords to VoiceCommand values
├── VoiceCommandResolver.java   — matches a transcript to a VoiceCommand
├── VoiceCommandExecutor.java   — executes the matched command in the app
├── VoiceInputManager.java      — wraps Android SpeechRecognizer
└── TtsManager.java             — wraps Android TextToSpeech
```

Plus three interfaces used by the executor to talk to other parts of the app:

```
com.fhdw.biot.speech.iot.mqtt/
└── IMqttPublisher.java         — lets executor publish MQTT commands

com.fhdw.biot.speech.iot.graph/
└── IFilterableChart.java       — lets executor apply date filters to charts

(placeholder for future development)
└── ILlmQueryHandler.java       — future LLM query interface (currently null)
```

---

## 3. Speech-to-Text: VoiceInputManager

### What is SpeechRecognizer?

`SpeechRecognizer` is a built-in Android class that converts audio from the microphone into text. It uses Google's speech recognition service when online, or a smaller on-device model when offline. It supports German and English out of the box.

> **Important distinction:** The phone microphone used for voice commands is completely separate from the KY-037 hardware microphone on the ESP8266. The KY-037 sends sound level data over MQTT. The phone mic is used by SpeechRecognizer to capture voice commands. These are two entirely independent systems.

### How VoiceInputManager works

`VoiceInputManager` wraps `SpeechRecognizer` to provide a clean push-to-talk interface without the standard Android popup dialog.

```java
// Starting a recognition session (called when user presses the voice button):
speechRecognizer.startListening(intent);

// The result comes back asynchronously:
@Override
public void onResults(Bundle results) {
    List<String> matches = results.getStringArrayList(
        SpeechRecognizer.RESULTS_RECOGNITION
    );
    if (matches != null && !matches.isEmpty()) {
        String transcript = matches.get(0);  // best match
        handleVoiceResult(transcript);
    }
}
```

Android returns a list of possible transcriptions ranked by confidence. The first one (`matches.get(0)`) is always the highest confidence result. That string is then passed into `handleVoiceResult()` in `MainActivity`, which feeds it to the resolver.

---

## 4. The Command Dictionary: VoiceCommandDictionary

`VoiceCommandDictionary` is where every supported command is defined as a set of keyword rules. It is a static lookup table, no machine learning, no fuzzy matching, just simple keyword checking.

### How the matching rules work

Each command has one or more **keyword groups**. A transcript must contain at least one word from each group for the command to match. Groups are ANDed together; words within a group are ORed.

**Example — navigating to the gyroscope screen:**

```java
// Rule: transcript must contain ("zeige" OR "öffne" OR "show" OR "open")
//       AND ("gyro" OR "gyroskop" OR "gyroscope")

VoiceCommand.NAVIGATE_GYRO, Arrays.asList(
    Arrays.asList("zeige", "öffne", "show", "open", "gehe", "navigate"),
    Arrays.asList("gyro", "gyroskop", "gyroscope")
)
```

So all of these would match:
- "zeige gyro"
- "öffne gyroskop"
- "show gyroscope"
- "navigate to gyro please"

But "zeige" alone would not match (missing second group), and "gyro test mode" without a navigation word would not match either.

**Example, setting stream mode:**

```java
VoiceCommand.SET_MODE_STREAM, Arrays.asList(
    Arrays.asList("stream", "streaming", "live", "kontinuierlich", "continuous"),
    Arrays.asList("modus", "mode", "setze", "set", "aktiviere", "enable")
)
```

### Why this approach?

- No internet connection required, all matching is local
- Works in both German and English
- Easy to add new commands by adding new entries
- Transparent, you can read exactly what words trigger what command
- Fast, just string contains checks, runs in microseconds

---

## 5. Intent Matching: VoiceCommandResolver

`VoiceCommandResolver` is a stateless class that takes a transcript string and returns the best matching `VoiceCommand`.

```java
public static VoiceCommand resolve(String transcript) {
    if (transcript == null || transcript.isEmpty()) {
        return VoiceCommand.UNKNOWN;
    }

    String lower = transcript.toLowerCase(Locale.ROOT);

    // Try every command in the dictionary
    for (Map.Entry<VoiceCommand, List<List<String>>> entry
             : VoiceCommandDictionary.getRules().entrySet()) {

        VoiceCommand command = entry.getKey();
        List<List<String>> groups = entry.getValue();

        boolean allGroupsMatch = true;

        // Every group must have at least one keyword present
        for (List<String> group : groups) {
            boolean groupMatches = false;
            for (String keyword : group) {
                if (lower.contains(keyword)) {
                    groupMatches = true;
                    break;
                }
            }
            if (!groupMatches) {
                allGroupsMatch = false;
                break;
            }
        }

        if (allGroupsMatch) {
            return command;  // first match wins
        }
    }

    return VoiceCommand.UNKNOWN;
}
```

The resolver has **no Android imports**. It only uses `java.util` and `java.lang`. This is intentional, it means you can write a plain JUnit test for it without spinning up an emulator:

```java
@Test
public void testGyroNavigation() {
    VoiceCommand result = VoiceCommandResolver.resolve("zeige gyroskop");
    assertEquals(VoiceCommand.NAVIGATE_GYRO, result);
}
```

---

## 6. All Available Commands: VoiceCommand enum

`VoiceCommand` is a Java enum, a fixed list of all possible things the user can ask for. Every value represents one distinct intent.

### Navigation commands

| Command | What it does |
|---------|-------------|
| `NAVIGATE_HOME` | Go to MainActivity (home screen) |
| `NAVIGATE_ACCEL` | Open AccelActivity (accelerometer charts) |
| `NAVIGATE_GYRO` | Open GyroActivity (gyroscope charts) |
| `NAVIGATE_MAGNET` | Open MagnetActivity (magnetometer charts) |
| `NAVIGATE_GRAPH` | Open MainGraphActivity (all sensors combined) |
| `NAVIGATE_EVENTS` | Open EreignisActivity (event log) |
| `NAVIGATE_SETTINGS` | Open SettingsActivity |

### Time filter commands

| Command | What it does |
|---------|-------------|
| `FILTER_LAST_10_MIN` | Apply 10-minute sliding window filter to current chart |
| `FILTER_LAST_HOUR` | Apply 1-hour filter |
| `FILTER_LAST_DAY` | Apply 24-hour filter |
| `FILTER_CLEAR` | Remove all active filters |

### MQTT mode commands

| Command | What it does |
|---------|-------------|
| `SET_MODE_STREAM` | Publish `STREAM` to `Control/Mode` |
| `SET_MODE_BURST` | Publish `BURST` to `Control/Mode` |
| `SET_MODE_AVERAGE` | Publish `AVERAGE` to `Control/Mode` |

### Combined sensor view commands

| Command | What it does |
|---------|-------------|
| `SHOW_ALL_SENSORS` | Navigate to graph view showing all sensors |
| `SHOW_SENSOR_SUMMARY` | Speak current values of all sensors via TTS |

### LLM query commands (Placeholder until LLM implemented)

| Command | What it does |
|---------|-------------|
| `LLM_QUERY` | Forward the full transcript to the LLM handler |
| `LLM_EXPLAIN_SENSOR` | Ask LLM to explain a sensor reading |
| `LLM_ANOMALY_CHECK` | Ask LLM to check for anomalies in current data |

### System commands

| Command | What it does |
|---------|-------------|
| `UNKNOWN` | No matching command found, TTS says "Ich habe das nicht verstanden" |
| `HELP` | TTS speaks a list of available commands |

---

## 7. Executing Actions: VoiceCommandExecutor

`VoiceCommandExecutor` receives a resolved `VoiceCommand` and carries it out. It holds references to everything it needs to act on:

```java
public class VoiceCommandExecutor {

    private final Activity activity;        // for navigation (startActivity)
    private final IMqttPublisher mqtt;      // for mode changes
    private final IFilterableChart chart;   // for date filters
    private final TtsManager tts;           // for spoken responses
    private final ILlmQueryHandler llm;     // null until Phase 4
```

The main entry point is `execute()`:

```java
public void execute(VoiceCommand command, String originalTranscript) {
    switch (command) {

        case NAVIGATE_GYRO:
            activity.startActivity(new Intent(activity, GyroActivity.class));
            tts.speak("Öffne Gyroskop");
            break;

        case SET_MODE_STREAM:
            mqtt.publish("Control/Mode", "STREAM", true);
            tts.speak("Stream Modus aktiviert");
            break;

        case FILTER_LAST_10_MIN:
            chart.applyTimeFilter(10);
            tts.speak("Zeige letzte 10 Minuten");
            break;

        case LLM_QUERY:
            if (llm != null) {
                llm.query(originalTranscript);
            } else {
                tts.speak("LLM noch nicht verfügbar");
            }
            break;

        case UNKNOWN:
            tts.speak("Ich habe das nicht verstanden. Sage Hilfe für eine Liste der Befehle.");
            break;

        // ... all other cases
    }
}
```

Every action speaks a confirmation via TTS so the user knows the command was understood, even without looking at the screen.

---

## 8. Text-to-Speech: TtsManager

`TtsManager` wraps Android's `TextToSpeech` class and adds a **pending utterance queue**. This solves a common problem: `TextToSpeech` initialises asynchronously, so if you call `speak()` before it is ready, the speech is silently dropped.

```java
public class TtsManager implements TextToSpeech.OnInitListener {

    private TextToSpeech tts;
    private boolean ready = false;
    private String pendingUtterance = null;  // queued speech

    public TtsManager(Context context) {
        tts = new TextToSpeech(context, this);
    }

    @Override
    public void onInit(int status) {
        if (status == TextToSpeech.SUCCESS) {
            tts.setLanguage(Locale.GERMAN);  // German voice
            ready = true;

            // Speak anything that was queued before we were ready
            if (pendingUtterance != null) {
                speak(pendingUtterance);
                pendingUtterance = null;
            }
        }
    }

    public void speak(String text) {
        if (ready) {
            tts.speak(text, TextToSpeech.QUEUE_FLUSH, null, null);
        } else {
            pendingUtterance = text;  // save it for when we're ready
        }
    }

    public void shutdown() {
        if (tts != null) {
            tts.stop();
            tts.shutdown();
        }
    }
}
```

### Language setting

`Locale.GERMAN` is used so the TTS voice speaks in German with correct pronunciation. If you want English responses, change this to `Locale.ENGLISH` or `Locale.US`.

### QUEUE_FLUSH vs QUEUE_ADD

`QUEUE_FLUSH` interrupts any currently speaking text and starts the new text immediately. If you wanted to queue multiple sentences to play sequentially without interruption, you would use `QUEUE_ADD` instead. For voice command confirmations `QUEUE_FLUSH` is correct, you always want the immediate response, not queued behind old speech.

---

## 9. The Three Interfaces

The executor uses three interfaces to communicate with other parts of the app without depending on concrete classes. This keeps the voice package loosely coupled.

### IMqttPublisher (in the mqtt package)

```java
public interface IMqttPublisher {
    void publish(String topic, String payload, boolean retained);
}
```

`MqttHandler` implements this interface. The executor calls it to publish mode changes to `Control/Mode`. Passing `MqttHandler` as an `IMqttPublisher` means the voice package does not need to import or know about `MqttHandler` directly.

### IFilterableChart (in the graph package)

```java
public interface IFilterableChart {
    void applyTimeFilter(int minutes);
    void clearFilter();
    boolean isFilterActive();
}
```

`AccelActivity`, `GyroActivity`, and `MagnetActivity` all implement this interface. When the user says "zeige letzte 10 Minuten", the executor calls `chart.applyTimeFilter(10)` on whichever activity is currently visible.

### ILlmQueryHandler (placeholder)

```java
public interface ILlmQueryHandler {
    void query(String transcript);
}
```

Currently passed as `null` in the executor constructor. This will be implemented, when the LLM_App integration is built. Any command that reaches `LLM_QUERY` currently results in a TTS message telling the user the LLM is not yet available.

---

## 10. Emulator vs Real Device

The voice system behaves differently depending on where the app is running:

| | Android Emulator | Real Android Phone |
|--|--|--|
| STT method | RecognizerIntent (shows dialog) | Background SpeechRecognizer (no popup) |
| Microphone source | Laptop microphone via emulator | Phone microphone directly |
| Language | Set by Google account locale | Set by Google account locale |
| TTS | Works identically | Works identically |
| Push-to-talk | Button triggers dialog | Button starts background listening |

Both paths feed into the same `handleVoiceResult()` method in `MainActivity`, so the resolution and execution logic is completely shared regardless of how the transcript was captured.

---

## 11. The Complete Flow: Step by Step

Here is exactly what happens when a user says "zeige gyroskop":

```
1. User presses the voice button in the app

2. VoiceInputManager.startListening() is called
   └── SpeechRecognizer begins capturing audio from phone mic

3. User says "zeige gyroskop"

4. SpeechRecognizer processes the audio
   └── Returns: ["zeige gyroskop", "zeige gyro", "zeige gyros"]
       (list ranked by confidence)

5. VoiceInputManager takes the first result: "zeige gyroskop"
   └── Calls MainActivity.handleVoiceResult("zeige gyroskop")

6. MainActivity calls VoiceCommandResolver.resolve("zeige gyroskop")
   └── Converts to lowercase: "zeige gyroskop"
   └── Checks VoiceCommandDictionary rules for NAVIGATE_GYRO:
       Group 1: ["zeige", "öffne", "show", ...] → "zeige" ✓
       Group 2: ["gyro", "gyroskop", "gyroscope"] → "gyroskop" ✓
       All groups match → return VoiceCommand.NAVIGATE_GYRO

7. MainActivity calls VoiceCommandExecutor.execute(NAVIGATE_GYRO, "zeige gyroskop")
   └── startActivity(new Intent(activity, GyroActivity.class))
   └── tts.speak("Öffne Gyroskop")

8. TtsManager.speak("Öffne Gyroskop")
   └── TextToSpeech engine speaks "Öffne Gyroskop" through the phone speaker

9. GyroActivity opens on screen
```

Total time from end of speech to screen navigation: under 500ms on a real device.

---

## 12. Adding New Commands

To add a new voice command, three files need to be updated:

**Step 1: Add the intent to `VoiceCommand.java`:**

```java
public enum VoiceCommand {
    // existing commands...
    NAVIGATE_GYRO,
    SET_MODE_STREAM,

    // your new command:
    SHOW_MIC_VALUE,
    UNKNOWN
}
```

**Step 2: Add the keyword rules to `VoiceCommandDictionary.java`:**

```java
put(VoiceCommand.SHOW_MIC_VALUE, Arrays.asList(
    Arrays.asList("zeige", "show", "was ist"),        // group 1: trigger words
    Arrays.asList("mikrofon", "mic", "microphone")    // group 2: subject words
));
```

**Step 3: Add the action to `VoiceCommandExecutor.java`:**

```java
case SHOW_MIC_VALUE:
    // read the current mic value from wherever it is stored
    tts.speak("Aktueller Mikrofonwert: " + currentMicValue);
    break;
```

That is everything. No other files need changing. The resolver and the input manager work automatically with any new command you add to the dictionary.

---
