# Events

## Purpose

Events represent user interactions or system triggers that the Screen sends to the ViewModel.

They flow in one direction: **Screen → ViewModel**.

---

## Definition

Events are a `sealed interface` nested inside the Contract:

```kotlin
sealed interface Event {
    // Text field changes
    data class OnNameChanged(val value: String) : Event
    data class OnLabelChanged(val value: String) : Event

    // Time/date pickers
    data class OnStartTimeSelected(val value: LocalTime) : Event
    data class OnEndTimeSelected(val value: LocalTime) : Event

    // Button taps
    data object OnSubmitClick : Event
    data object OnCancelClick : Event

    // Dialog interactions
    data object OnDialogDismissed : Event

    // Lifecycle triggers
    data object LoadData : Event
}
```

---

## Naming Conventions

| Interaction | Pattern | Example |
|-------------|---------|---------|
| Text changed | `OnXxxChanged(val value: String)` | `OnNameChanged` |
| Button tapped | `OnXxxClick` | `OnSubmitClick` |
| Item tapped | `OnXxxTapped(val id: String)` | `OnStudentTapped` |
| Item selected | `OnXxxSelected(val item: T)` | `OnStartTimeSelected` |
| Dialog dismissed | `OnDialogDismissed` | |
| Toggle switched | `OnXxxToggled` | `OnNotificationsToggled` |
| Screen opens | `LoadData` | |

---

## How the Screen Sends Events

```kotlin
// Method reference — clean for single-event composables
XxxBody(onEvent = viewModel::event)

// Lambda — clear for specific events
PkButton(onClick = { viewModel.event(Event.OnSubmitClick) })

// Inside a text field
OutlinedTextField(
    onValueChange = { viewModel.event(Event.OnNameChanged(it)) }
)
```

---

## How the ViewModel Handles Events

```kotlin
override fun event(event: Event) {
    when (event) {
        is Event.OnNameChanged -> validateName(event.value)
        is Event.OnLabelChanged -> validateLabel(event.value)
        is Event.OnStartTimeSelected -> updateStartTime(event.value)
        is Event.OnEndTimeSelected -> updateEndTime(event.value)
        Event.OnSubmitClick -> submit()
        Event.OnCancelClick -> sendEffect(Effect.NavigateBack)
        Event.OnDialogDismissed -> dismissDialog()
        Event.LoadData -> loadData()
    }
}
```

---

## Rules

Always:

- Name events after **what the user did**, not what the ViewModel should do
- Use `data object` for events with no payload
- Use `data class` for events carrying data

Never:

- Put logic inside an event — events are pure data
- Call ViewModel methods from Composables without going through `event()`
- Create events that combine multiple interactions — one interaction, one event
