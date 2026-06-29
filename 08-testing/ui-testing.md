# UI Testing

## Strategy

Compose UI tests verify that the screen renders the correct UI for a given `UiState`.

Because Screens are stateless composables that receive state as parameters, they are straightforward to test:
provide a `UiState` → assert on what is shown.

---

## Test Setup

```kotlin
// androidTest/feature/groups/GroupScreenTest.kt
@RunWith(AndroidJUnit4::class)
class GroupScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `when enabledButton is false, save button is disabled`() {
        composeTestRule.setContent {
            AppTheme {
                GroupBody(
                    uiState = GroupContract.UiState(
                        enabledButton = false,
                        screenUiModel = GroupContract.GroupScreenUiModel(
                            saveButtonText = "Save",
                        )
                    ),
                    onEvent = {},
                )
            }
        }

        composeTestRule
            .onNodeWithText("Save")
            .assertIsNotEnabled()
    }

    @Test
    fun `when nameError is set, error text is shown`() {
        composeTestRule.setContent {
            AppTheme {
                GroupBody(
                    uiState = GroupContract.UiState(
                        groupName = "",
                        groupNameError = "Name is required",
                    ),
                    onEvent = {},
                )
            }
        }

        composeTestRule
            .onNodeWithText("Name is required")
            .assertIsDisplayed()
    }

    @Test
    fun `typing in the name field sends OnNameChanged event`() {
        val events = mutableListOf<GroupContract.Event>()

        composeTestRule.setContent {
            AppTheme {
                GroupBody(
                    uiState = GroupContract.UiState(),
                    onEvent = { events += it },
                )
            }
        }

        composeTestRule
            .onNode(hasSetTextAction())
            .performTextInput("Algebra I")

        val nameEvents = events.filterIsInstance<GroupContract.Event.OnNameChanged>()
        assertTrue(nameEvents.isNotEmpty())
        assertEquals("Algebra I", nameEvents.last().value)
    }

    @Test
    fun `when dialogState is Loading, loading dialog is shown`() {
        composeTestRule.setContent {
            AppTheme {
                GroupDialogState(
                    uiState = GroupContract.UiState(
                        dialogState = GroupContract.DialogState.Loading,
                        screenUiModel = GroupContract.GroupScreenUiModel(
                            dialogLoadingTitle = "Saving..."
                        )
                    ),
                    onDismiss = {},
                )
            }
        }

        composeTestRule
            .onNodeWithText("Saving...")
            .assertIsDisplayed()
    }
}
```

---

## Common Matchers

```kotlin
// Find by text
onNodeWithText("Save")
onNodeWithText("Name is required")

// Find by content description (accessibility)
onNodeWithContentDescription("Close")

// Find by semantic
onNode(hasSetTextAction())           // text input field
onNode(hasClickAction())             // any clickable element
onNode(isEnabled())                  // enabled state
onNode(isNotEnabled())               // disabled state

// Assert
.assertIsDisplayed()
.assertIsNotDisplayed()
.assertIsEnabled()
.assertIsNotEnabled()
.assertTextEquals("text")

// Interact
.performClick()
.performTextInput("text")
.performScrollTo()
```

---

## Test Strategy per Component Type

| Test target | Approach |
|-------------|----------|
| Screen body (`XxxBody`) | Pass `UiState` directly, assert rendered output |
| Dialog composable | Pass `DialogState`, assert visibility and content |
| Reusable component | Isolated test — pass only the component's params |
| Full screen with ViewModel | Integration test — use `HiltAndroidTest` |

---

## Rules

Always:

- Test stateless composables — pass `UiState`, not a ViewModel
- Wrap the composable under test in `AppTheme { }`
- Use `createComposeRule()` — not `createAndroidComposeRule<Activity>()` for unit-level UI tests

Never:

- Test the ViewModel indirectly through the UI in a unit test
- Use `Thread.sleep` in UI tests — use `composeTestRule.waitForIdle()`
- Assert on exact colors or sizes — assert on semantic meaning (enabled, visible, text)
