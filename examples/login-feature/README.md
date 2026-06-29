# Example: Login Feature

Complete walkthrough of a login feature with email + password, validation, and auth error handling.

---

## File Structure

```
presentation/feature/login/
├── LoginRouter.kt
├── LoginNavGraph.kt
├── contract/
│   └── LoginContract.kt
├── viewModel/
│   └── LoginViewModel.kt
└── ui/
    └── LoginScreen.kt

domain/usecases/
└── LoginUseCase.kt

domain/repository/
└── AuthRepository.kt

data/repository/
└── AuthRepositoryImpl.kt
```

---

## Step 1: Router

```kotlin
// presentation/feature/login/LoginRouter.kt
object LoginRouter {
    const val ROUTE = "login"
}
```

---

## Step 2: Contract

```kotlin
// presentation/feature/login/contract/LoginContract.kt
interface LoginContract : MVIContract<LoginContract.UiState, LoginContract.Effect, LoginContract.Event> {

    data class UiState(
        val email: String = "",
        val emailError: String? = null,
        val password: String = "",
        val passwordError: String? = null,
        val enabledButton: Boolean = false,
        val screenUiModel: LoginScreenUiModel = LoginScreenUiModel(),
        val dialogState: DialogState = DialogState.None,
    )

    data class LoginScreenUiModel(
        val toolbarTitle: String = "",
        val emailLabel: String = "",
        val passwordLabel: String = "",
        val loginButtonText: String = "",
        val dialogLoadingTitle: String = "",
        val dialogLoadingMessage: String = "",
        val dialogErrorTitle: String = "",
        val dialogConfirmButton: String = "",
    )

    sealed interface Effect {
        data object NavigateToHome : Effect
    }

    sealed interface Event {
        data class OnEmailChanged(val value: String) : Event
        data class OnPasswordChanged(val value: String) : Event
        data object OnLoginClick : Event
        data object OnDialogDismissed : Event
    }

    sealed class DialogState {
        data object None : DialogState()
        data object Loading : DialogState()
        data class Error(val message: String) : DialogState()
    }
}
```

---

## Step 3: ViewModel

```kotlin
// presentation/feature/login/viewModel/LoginViewModel.kt
@HiltViewModel
class LoginViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val loginUseCase: LoginUseCase,
) : ViewModel(), LoginContract {

    private val _state = MutableStateFlow(LoginContract.UiState())
    override val state: StateFlow<LoginContract.UiState> = _state.asStateFlow()

    private val _effect = MutableSharedFlow<LoginContract.Effect>()
    override val effect: SharedFlow<LoginContract.Effect> = _effect.asSharedFlow()

    init {
        updateState { copy(screenUiModel = createUiStrings()) }
    }

    override fun event(event: LoginContract.Event) {
        when (event) {
            is LoginContract.Event.OnEmailChanged -> updateState {
                copy(
                    email = event.value,
                    emailError = null,
                ).recalculateFormValidity()
            }
            is LoginContract.Event.OnPasswordChanged -> updateState {
                copy(
                    password = event.value,
                    passwordError = null,
                ).recalculateFormValidity()
            }
            is LoginContract.Event.OnLoginClick -> login()
            is LoginContract.Event.OnDialogDismissed -> updateState {
                copy(dialogState = LoginContract.DialogState.None)
            }
        }
    }

    private fun login() {
        updateState { copy(dialogState = LoginContract.DialogState.Loading) }
        viewModelScope.launch {
            loginUseCase(email = _state.value.email, password = _state.value.password)
                .fold(
                    onSuccess = {
                        sendEffect(LoginContract.Effect.NavigateToHome)
                    },
                    onError = { failure ->
                        updateState {
                            copy(dialogState = LoginContract.DialogState.Error(mapFailureToMessage(failure)))
                        }
                    },
                )
        }
    }

    private fun mapFailureToMessage(failure: Failure): String = when (failure) {
        is Failure.Unauthorized -> context.getString(R.string.error_invalid_credentials)
        is Failure.NetworkError -> context.getString(R.string.error_network)
        else -> context.getString(R.string.error_unknown)
    }

    private fun createUiStrings() = LoginContract.LoginScreenUiModel(
        toolbarTitle = context.getString(R.string.login_title),
        emailLabel = context.getString(R.string.login_email_label),
        passwordLabel = context.getString(R.string.login_password_label),
        loginButtonText = context.getString(R.string.login_button),
        dialogLoadingTitle = context.getString(R.string.login_loading_title),
        dialogLoadingMessage = context.getString(R.string.login_loading_message),
        dialogErrorTitle = context.getString(R.string.login_error_title),
        dialogConfirmButton = context.getString(R.string.dialog_ok),
    )

    private fun updateState(update: LoginContract.UiState.() -> LoginContract.UiState) {
        _state.update { it.update() }
    }

    private fun sendEffect(effect: LoginContract.Effect) {
        viewModelScope.launch { _effect.emit(effect) }
    }
}

private fun LoginContract.UiState.recalculateFormValidity(): LoginContract.UiState {
    val emailValid = validateRequiredTextField(email)
    val passwordValid = validateRequiredTextField(password)
    return copy(enabledButton = emailValid && passwordValid)
}
```

---

## Step 4: UseCase

```kotlin
// domain/usecases/LoginUseCase.kt
class LoginUseCase @Inject constructor(
    private val authRepository: AuthRepository,
) {
    suspend operator fun invoke(email: String, password: String): Either<Failure, Unit> {
        if (email.isBlank()) return Either.Error(Failure.EmptyData)
        if (password.length < 8) return Either.Error(Failure.UnknownError)
        return authRepository.login(email = email, password = password)
    }
}
```

---

## Step 5: Repository

```kotlin
// domain/repository/AuthRepository.kt
interface AuthRepository {
    suspend fun login(email: String, password: String): Either<Failure, Unit>
}

// data/repository/AuthRepositoryImpl.kt
class AuthRepositoryImpl @Inject constructor(
    private val apiService: AuthApiService,
) : AuthRepository {

    override suspend fun login(email: String, password: String): Either<Failure, Unit> =
        safeApiCall { apiService.login(LoginRequest(email, password)) }
}
```

---

## Step 6: Screen

```kotlin
// presentation/feature/login/ui/LoginScreen.kt
@Composable
fun LoginScreen(
    onNavigateToHome: () -> Unit,
    viewModel: LoginViewModel = hiltViewModel(),
) {
    val uiState by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is LoginContract.Effect.NavigateToHome -> onNavigateToHome()
            }
        }
    }

    LoginBody(uiState = uiState, onEvent = viewModel::event)
}

@Composable
private fun LoginBody(
    uiState: LoginContract.UiState,
    onEvent: (LoginContract.Event) -> Unit,
) {
    Scaffold(
        topBar = { AppToolbar(title = uiState.screenUiModel.toolbarTitle) }
    ) { padding ->
        LoginFormContent(
            uiState = uiState,
            onEvent = onEvent,
            modifier = Modifier.padding(padding),
        )
    }
    LoginDialogState(uiState = uiState, onEvent = onEvent)
}

@Composable
private fun LoginFormContent(
    uiState: LoginContract.UiState,
    onEvent: (LoginContract.Event) -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(SpacingL),
        verticalArrangement = Arrangement.spacedBy(SpacingM),
    ) {
        AppTextField(
            value = uiState.email,
            onValueChange = { onEvent(LoginContract.Event.OnEmailChanged(it)) },
            label = uiState.screenUiModel.emailLabel,
            errorText = uiState.emailError,
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
        )
        AppTextField(
            value = uiState.password,
            onValueChange = { onEvent(LoginContract.Event.OnPasswordChanged(it)) },
            label = uiState.screenUiModel.passwordLabel,
            errorText = uiState.passwordError,
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
        )
        Spacer(modifier = Modifier.weight(1f))
        AppButton(
            text = uiState.screenUiModel.loginButtonText,
            onClick = { onEvent(LoginContract.Event.OnLoginClick) },
            enabled = uiState.enabledButton,
        )
    }
}

@Composable
private fun LoginDialogState(
    uiState: LoginContract.UiState,
    onEvent: (LoginContract.Event) -> Unit,
) {
    when (val dialog = uiState.dialogState) {
        is LoginContract.DialogState.Loading -> AppStatusDialog(
            title = uiState.screenUiModel.dialogLoadingTitle,
            message = uiState.screenUiModel.dialogLoadingMessage,
            onDismiss = {},
            isLoading = true,
        )
        is LoginContract.DialogState.Error -> AppStatusDialog(
            title = uiState.screenUiModel.dialogErrorTitle,
            message = dialog.message,
            onDismiss = { onEvent(LoginContract.Event.OnDialogDismissed) },
            confirmButtonText = uiState.screenUiModel.dialogConfirmButton,
        )
        is LoginContract.DialogState.None -> Unit
    }
}
```

---

## Step 7: Test

```kotlin
class LoginViewModelTest : BaseTest() {

    private val fakeAuthRepository = FakeAuthRepository()
    private lateinit var viewModel: LoginViewModel

    @Before
    fun setup() {
        val context = mockk<Context>(relaxed = true)
        viewModel = LoginViewModel(context, LoginUseCase(fakeAuthRepository))
    }

    @Test
    fun `initial state has empty fields and disabled button`() = runTest {
        assertFalse(viewModel.state.value.enabledButton)
        assertEquals("", viewModel.state.value.email)
    }

    @Test
    fun `typing email and password enables the button`() = runTest {
        viewModel.event(LoginContract.Event.OnEmailChanged("user@example.com"))
        viewModel.event(LoginContract.Event.OnPasswordChanged("password123"))
        assertTrue(viewModel.state.value.enabledButton)
    }

    @Test
    fun `successful login emits NavigateToHome effect`() = runTest {
        val effects = mutableListOf<LoginContract.Effect>()
        backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) {
            viewModel.effect.toList(effects)
        }

        viewModel.event(LoginContract.Event.OnEmailChanged("user@example.com"))
        viewModel.event(LoginContract.Event.OnPasswordChanged("password123"))
        viewModel.event(LoginContract.Event.OnLoginClick)

        assertContains(effects, LoginContract.Effect.NavigateToHome)
    }

    @Test
    fun `failed login shows error dialog`() = runTest {
        fakeAuthRepository.shouldFail = true

        viewModel.event(LoginContract.Event.OnEmailChanged("user@example.com"))
        viewModel.event(LoginContract.Event.OnPasswordChanged("password123"))
        viewModel.event(LoginContract.Event.OnLoginClick)

        assertIs<LoginContract.DialogState.Error>(viewModel.state.value.dialogState)
    }
}
```
