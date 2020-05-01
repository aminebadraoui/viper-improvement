### The Viper architecture:
Interactor, presenter (output interface, basic event handler), userInterface, wireframe.
Interactor implements interactorInterface and has reference to the interactorOutputInterface.
Presenter implements the interactorOutputInterface and the eventHandler interface and has a reference to the interactorInterface and to the userInterface.
The view implements the userInterface protocol and has a reference to the event handler interface. 

This respects the clean code paradigm since: 
The High level modules don’t depend on lower level modules, and both of them depend on abstractions. 
There is clear separation of concerns. (which makes it easier to test)

**The trade-off with Viper:**

A lot of boilerplate, and the high level logic of how the components interact with each other is the same across the modules, it’s only the “implementation” (the contract methods) that differ due to the specificity of each module. 
Fortunately, when even the implemention is the same between modules (loading state, showing empty state …), we use inheritance thus preventing from repeating ourselves and making potential mistakes on the way. 

### Inheritence: 

**The trade-off with inheritence:** 

* **Overhead**: The base classes have a lot of functionalities that might not be needed. (example dock panel configuration for a screen that doesn’t need a dock panel)

* **Rigidity**: While some level of rigidity might be good to get a level of consistency across the app, the trade off is that adding a custom behavior might get problematic. 

### Motivation: 
Minimize the trade offs of viper and inheritance.
Make the test even more testable. 
Alleviate the tech debt of the legacy code from the evolving codebase. 

Code duplication: There is some code duplication because of codebase growth (there is a new way of doing things and an old way that is still used in the app, which makes it a bit confusing for a new developer who jumps in the project)

All of this leads to unpredictable behavior in the code. 

## Proposed Solution: A more unified viper and favoring composition over inheritance 
As a proof of concept I tried to replicate the settings screen because it is fairly simple: A tableView with multiple sections. 

### A unified Viper:  One method (signature) to rule them all

Depending the situation/module, the actions to perform might be different (hence the module contract) but the behavior and separation of concern  the same. 

That means we can abstract away the logic behind how the viper components communicate. Hence the suggestion of having a function called handle(_ event: EventType) where the EventType is a type-erased enun representing what kind of event the viper component needs to handle. 

Now all viper components across all the modules can have the same the handle function signature, and the implementation of that function is left to the module that needs to handle it. This can be done simply by switching on the cases of that event enum and perform the appropriate action.

**Naming convention:** 

To make the responsibilities of each viper component more explicit we can swap the input/output terminology with the listener terminology so it is clear what is expected to be received in that module. 

Furthermore, we could refer to the “messages” sent from the view to the interactor as events; and from the interactor to the view as commands. 

With those two premises in place, here is the suggested naming convention: 

The event handler, implemented by the presenter, can be called the ViewEventListener.

The interactor interface can be called the presenterEventListener. 

The interactor output  interface can be called the interactorCommandListener.

The user interface can be called the presenterCommandListener. 

All those protocols are conformed by higher level protocols: ViewProtocol, PresenterProtocol and InteractorProtocol.
**Example in the POC:** 

``` swift
protocol SettingsViewProtocol: ViewProtocol {}
protocol SettingsPresenterProtocol: PresenterProtocol {}
protocol SettingsInteractorProtocol: InteractorProtcol {}
```

lets take the example of the ViewProtocol: 

 It listens to commands from the presenter so it implements  the PresenterCommandListener interface. And it sends view events to a viewEventListener so it needs a reference to the latter. 

The associatedType allows to erase the type of the event/command and specify it later with the concrete enum used by the specific module.

**Note:** The AnyEventListener class is workaround because so we can’t put a variable that conforms to a generic protocol inside another protocol. So we need to go through a generic class that I called AnyEventListener.  

```swift
protocol ViewProtocol: class, PresenterCommandListener  {
    associatedtype ViewEventType: ViewEvent
    
    var eventListener: AnyEventListener<ViewEventType>? { get set }
}
```

The Presenter Command Listener, like all the other listeners has a handle function. 

```swift
protocol PresenterCommandListener {
    associatedtype PresenterCommandType: PresenterCommand

    func handle(_ command: PresenterCommandType)
}
```

Now let’s take a look at how this is handled in the Settings Module: 

Here are some examples of events (from the interactor and from the presenter) 

```swift
enum SettingsInteractorResponse: InteractorResponse {
    case
    startLoading,
    endLoading,
    presentSettings,
    error(error: Error)
}

enum SettingsPresenterCommand: PresenterCommand {
    case
    showLoading,
    hideLoading,
    displayTable(dataSource: [TableSection]),
    showError(error: Error)
}
```

The presenter handles the interactor events as follows: 

```swift
    func handle(_ response: SettingsInteractorResponse) {
        switch response {
        case .startLoading:
            presenterCommandListener?.handle(.showLoading)
        case .presentSettings:
            presentTable()
        case .error(let error):
            presenterCommandListener?.handle(.showError(error: error))
        case .endLoading:
            presenterCommandListener?.handle(.hideLoading)
        }
    }
```

Let’s see the first case, when we get a startLoading event from the interactor, we handle it by calling our presenter command listener and send the showLoading event. 

The presenter command listener is the view controller, so let’s see how it handles this command.

As we saw earlier, the view is a presenter commands listener, so it has a handle function that accepts presenter commands, in this case the SettingsPresenterCommand enum:
    
```swift
    func handle(_ command: SettingsPresenterCommand) {
        switch command {
        case .showLoading:
            startLoadingAnimation()
        case .hideLoading:
            stopLoadingAnimation()
        case .showError(let error):
            show(error)
        case .displayTable(let dataSource):
            displayTableViewModel(from: dataSource)
        }
    }
```

When receiving the show loading command it runs the startLoadingAnimation() function. 

But where is the startLoadingAnimation function coming from? 

### Protocol composition 

Before:

```swift
protocol SettingsViewProtocol: ViewProtocol, LoadingDisplayable
```

Here is the LoadingDisplayable protocol: 

```swift
protocol LoadingDisplayable {
var waitingView: WaitingView { get } 

    func startLoadingAnimation()
    func stopLoadingAnimation()
}
```

But to avoid to implement those functions everywhere, we use the power of conditional conformance and protocol extensions: 

```swift
extension LoadingDisplayable where Self: UIViewController {

    func showWaitingView() {
        if waitingView.superview == nil {
            waitingView.alpha = 0
            view.addSubview(waitingView)
            waitingView.pinAllSidesToSuperview()
        }
			// Rest of implementation ...
    }
    
    func hideWaitingView() {
        containerView.isUserInteractionEnabled = true
        waitingView.stopAnimating()
			// Rest of implentation ...
    }
} 
```
