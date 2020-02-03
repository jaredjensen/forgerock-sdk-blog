# Using Authentication Tree Stage to Build a Custom UI with the ForgeRock JavaScript SDK

The [ForgeRock JavaScript SDK](https://sdks.forgerock.com/javascript/index/) greatly simplifies adding intelligent authentication to your applications. It offers a ready-to-use UI that completely handles rendering of authentication steps. You can also take full control and create a custom UI, in which case it's helpful to know the current _stage_ of the authentication tree to determine which UI you should render.

OpenAM 7.0 adds a _stage_ property to the _page node_ that can be used for this purpose, and alternate approaches are available for prior versions. This post will illustrate three options.

## OpenAM 6.5

While the _stage_ property doesn't exist in authentication trees prior to OpenAM 7.0, there are two alternate approaches to achieve the same result.

### Approach #1: Metadata Callbacks

Using metadata callbacks, you can inject the _stage_ value into the tree's response payload. The only difference is that the value will appear in a callback, instead of directly associated with the step itself.

This approach involves three steps:

1. Create a script to add the metadata callback
1. Update your tree to execute that script
1. Read the metadata callback in your application

#### Step 1: Add a Metadata Callback using a Script

Create a script of type _Decision node script for authentication trees_. Give it an appropriate name, such as "MetadataCallback: UsernamePassword". In the script, add a metadata callback that creates an object with a `stage` property. Be sure to also set the `outcome` value.

```js
var fr = JavaImporter(
  org.forgerock.json.JsonValue,
  org.forgerock.openam.auth.node.api.Action,
  com.sun.identity.authentication.spi.MetadataCallback
);

with (fr) {
  var json = JsonValue.json({ stage: "UsernamePassword" });
  action = Action.send(new MetadataCallback(json)).build();
}

outcome = "true";
```

As with all scripts, ensure you have whitelisted any imported classes.

#### Step 2: Update your Tree to Execute Script

Add a _scripted decision node_ to your _page node_ and configure it to reference the script created in the previous step. In this example, the step payload will contain three callbacks:

- MetadataCallback
- NameCallback
- PasswordCallback

![Scripted Decision Node to add stage](img/scripted_decision_node.png)

#### Step 3: Read the Metadata Callback

Now use the SDK to find the metadata callback and read its _stage_ property.

```js
function getStage(step) {
  // Get all metadata callbacks in the step
  const metadataCallbacks = step.getCallbacksOfType(CallbackType.MetadataCallback);

  // Find the first callback that contains a "stage" value in its data
  const stage = metadataCallbacks
    .map(x => {
      const data = x.getData();
      const dataIsObject = typeof data === "object" && data !== null;
      return dataIsObject && data.stage ? data.stage : undefined;
    })
    .find(x => x !== undefined);

  // Return the stage value, which will be undefined if none exists
  return stage;
}
```

### Approach #2: Inspecting Callbacks

If you have relatively few and/or well-known authentication trees, it's likely you can determine the _stage_ by simply looking at the types of callbacks present in the step.

For example, it's common for a tree to start by capturing username and password. In this case, you can inspect the callbacks to see if they consist of a _NameCallback_ and _PasswordCallback_. If your tree uses WebAuthn for passwordless authentication, the SDK can help with this inspection.

```js
function getStage(step) {
  // Check if the step contains callbacks for capturing username and password
  const usernameCallbacks = step.getCallbacksOfType(CallbackType.NameCallback);
  const passwordCallbacks = step.getCallbacksOfType(CallbackType.PasswordCallback);
  if (usernameCallbacks.length > 0 && passwordCallbacks.length > 0) {
    return "UsernamePassword";
  }

  // Use the SDK to determine if this is a WebAuthn step
  const webAuthnStepType = FRWebAuthn.getWebAuthnStepType(step);
  if (webAuthnStepType === WebAuthnStepType.Authentication) {
    return "DeviceAuthentication";
  } else if (webAuthnStepType === WebAuthnStepType.Registration) {
    return "DeviceRegistration";
  }

  // ... Add checks to determine other stages in your trees ...

  return undefined;
}
```

## OpenAM 7.0

Using OpenAM 7.0 to specify _stage_ is straightforward. When constructing a tree, place nodes inside a _page node_ and then specify its _stage_, which is a free-form text field.

![Stage property in OpenAM 7.0 trees](img/am7_editor_stage.png)

When you use the SDK's `FRAuth` module to iterate through a tree, you can now call the `getStage()` method on the returned `FRStep` and decide what custom UI needs to be rendered.

```ts
// Get the current step in the tree
const currentStep = await FRAuth.next(previousStep);

// Use the stage value configured in the tree
switch (currentStep.getStage()) {
  case "UsernamePassword":
    // Render your custom username/password UI
    break;
  case "SomeOtherStage":
    // etc
    break;
}
```
