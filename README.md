# Flutter Connectivity Wrapper Documentation

## Overview

This documentation explains how to implement and use the `ConnectivityWrapper` system in your Flutter application to handle internet connectivity status. The system provides:

1. Real-time network connectivity monitoring
2. Automatic display of a "No Internet" screen when offline
3. A loading state during connectivity checks
4. Localization support for error messages
5. Manual refresh capability

## Implementation Steps

### 1. Wrap Your Main App

Wrap your `MaterialApp` with the `ConnectivityWrapper` in your main application widget:

```dart
class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(langProvider);
    SizeConfig.init(context);
    return ConnectivityWrapper(
      child: MaterialApp(
        locale: locale,
        debugShowCheckedModeBanner: false,
        title: 'TalMal App',
        theme: TalmalTheme.light,
        initialRoute: AppRoutes.splash,
        onGenerateRoute: AppRoutes.generateRoute,
        supportedLocales: const [Locale('en'), Locale('ar')],
        localizationsDelegates: const [
          AppLocalization.delegate,
          GlobalMaterialLocalizations.delegate,
          GlobalWidgetsLocalizations.delegate,
          GlobalCupertinoLocalizations.delegate,
        ],
        localeResolutionCallback: (deviceLocale, supportedLocales) {
          for (var supportedLocale in supportedLocales) {
            if (deviceLocale != null &&
                deviceLocale.languageCode == supportedLocale.languageCode) {
              return deviceLocale;
            }
          }
          return Locale('ar');
        },
      ),
    );
  }
}
```

### 2. Required Dependencies

Add these dependencies to your `pubspec.yaml`:

```yaml
dependencies:
  connectivity_plus: ^4.0.2
  internet_connection_checker: ^1.0.0
  flutter_riverpod: ^2.4.9
```

### 3. Connectivity Provider

The system uses a Riverpod provider to manage connectivity state:

```dart
enum ConnectivityStatus {
  connected,
  disconnected,
  checking,
}

class ConnectivityNotifier extends StateNotifier<ConnectivityStatus> {
  ConnectivityNotifier() : super(ConnectivityStatus.checking) {
    _initConnectivity();
    _connectivitySubscription = _connectivity.onConnectivityChanged.listen(_updateConnectionStatus);
  }

  final Connectivity _connectivity = Connectivity();
  late StreamSubscription<ConnectivityResult> _connectivitySubscription;

  Future<void> _initConnectivity() async {
    try {
      final result = await _connectivity.checkConnectivity();
      await _updateConnectionStatus(result);
    } catch (e) {
      state = ConnectivityStatus.disconnected;
    }
  }

  Future<void> _updateConnectionStatus(ConnectivityResult result) async {
    if (result == ConnectivityResult.none) {
      state = ConnectivityStatus.disconnected;
    } else {
      final hasInternet = await InternetConnectionChecker().hasConnection;
      state = hasInternet ? ConnectivityStatus.connected : ConnectivityStatus.disconnected;
    }
  }

  Future<void> checkConnectivity() async {
    state = ConnectivityStatus.checking;
    await _initConnectivity();
  }

  @override
  void dispose() {
    _connectivitySubscription.cancel();
    super.dispose();
  }
}

final connectivityProvider = StateNotifierProvider<ConnectivityNotifier, ConnectivityStatus>(
      (ref) => ConnectivityNotifier(),
);
```

## Usage

### Checking Connectivity Status

You can check the current connectivity status anywhere in your app:

```dart
final connectivityStatus = ref.watch(connectivityProvider);

if (connectivityStatus == ConnectivityStatus.connected) {
  // You have internet
} else if (connectivityStatus == ConnectivityStatus.disconnected) {
  // No internet
} else {
  // Checking status
}
```

### Manually Refreshing Connectivity

To manually check connectivity (like in a retry button):

```dart
ref.read(connectivityProvider.notifier).checkConnectivity();
```

### No Internet Screen

The system automatically shows a localized "No Internet" screen when offline. The screen includes:

1. An offline icon
2. Localized title and description
3. A retry button

## Customization

### 1. Customizing the No Internet Screen

Modify the `NoInternetScreen` widget to match your app's design:

```dart
class NoInternetScreen extends ConsumerWidget {
  const NoInternetScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(langProvider);

    return Scaffold(
      backgroundColor: Colors.white,
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              // Customize the icon
              Container(
                width: 120,
                height: 120,
                decoration: BoxDecoration(
                  color: Colors.grey.shade100,
                  shape: BoxShape.circle,
                ),
                child: Icon(
                  Icons.wifi_off_rounded,
                  size: 60,
                  color: TalmalColors.red,
                ),
              ),

              const SizedBox(height: 32),

              // Customize the title
              Text(
                _getLocalizedText(context, 'no_internet_title', 'No Internet Connection'),
                style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                  fontWeight: FontWeight.bold,
                  color: Colors.grey.shade800,
                ),
                textAlign: TextAlign.center,
              ),

              // ... rest of the implementation
            ],
          ),
        ),
      ),
    );
  }
}
```

### 2. Localization

Add these keys to your localization files:

```arb
// English
{
  "no_internet_title": "No Internet Connection",
  "no_internet_description": "Please check your internet connection and try again. Make sure you are connected to Wi-Fi or mobile data.",
  "try_again_button": "Try Again"
}

// Arabic
{
  "no_internet_title": "لا يوجد اتصال بالإنترنت",
  "no_internet_description": "يرجى التحقق من اتصالك بالإنترنت والمحاولة مرة أخرى. تأكد من أنك متصل بشبكة Wi-Fi أو بيانات الجوال.",
  "try_again_button": "حاول مرة أخرى"
}
```

## Technical Notes

1. The system checks both network connectivity and actual internet access
2. It handles both WiFi and mobile data connections
3. The wrapper preserves your app's localization context
4. The loading overlay appears during connectivity checks
5. All network checks are performed asynchronously

## Best Practices

1. Use the connectivity status to disable network-dependent features when offline
2. Consider adding a "retry" button in your error screens that calls `checkConnectivity()`
3. Test with different network conditions (WiFi, mobile data, airplane mode)
4. Ensure your localization files contain the required keys
