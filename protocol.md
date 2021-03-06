# Language Server Protocol 

This document describes version 2.x of the language server protocol. Changes are marked with a change or new bar on the left hand side. Below is a summary of the major changes compared to version 1.0:

- consistent use of properties on request param objects. These properies are now structured like parameters in API's. So for example if an API takes a text document as the first parameter the corresponding parameter literal in the JSON RPC protocol now has a property `textDocument`. 
- consistent support for language identifiers. This means that the language ID is passed to the server via the open notification. Further references to the text document don't transfer this informaiton anymore.
- support for version numbers on documents. The initial version number is send to the server via the open notification. Document change notification do carry information about version numbers as well.
- Support for request cancellation.
- Interface naming: using consinstent names without I prefix.

The 1.x version of this document can be found [here](https://github.com/Microsoft/language-server-protocol/blob/master/versions/protocol-1-x.md)

## Base Protocol

The base protocol consists of a header and a content part (comparable to http). The header and content part are
separated by a '\r\n'.

### Header Part

The header part consists of header fields. Each header field is comprised of a name and a value,
separated by `: ` (a colon and a space).
Each header field is terminated by '\r\n'. 
Considering the last header field and the overall header itself are each terminated with '\r\n',
and that at least one header is mandatory, this means that two '\r\n' sequences always 
immediately precede the content part of a message.

Currently the following header fields are supported:

| Header File Name | Value Type  | Description |
|:-----------------|:------------|:------------|
| Content-Length   | number      | The length of the content part in bytes. This header is required. |
| Content-Type     | string      | The mime type of the content part. Defaults to application/vscode-jsonrpc; charset=utf8 |

The header part is encoded using the 'ascii' encoding. This includes the '\r\n' separating the header and content part.

### Content Part

Contains the actual content of the message. The content part of a message uses [JSON-RPC](http://www.jsonrpc.org/) to describe requests, responses and notifications. The content part is encoded using the charset provided in the Content-Type field. It defaults to 'utf8' which is the only encoding supported right now. 


### Example:

```
Content-Length: ...\r\n
\r\n
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "textDocument/didOpen", 
	"params": {
		...
	}
}
```
### Base Protocol JSON structures

The following TypeScript definitions describe the base JSON-RPC protocol:

#### Abstract Message

A general message as defined by JSON-RPC. The language server protocol always uses "2.0" as the jsonrpc version.

```typescript
interface Message {
	jsonrpc: string;
}
```
#### RequestMessage 

A request message to decribe a request between the client and the server. Every processed request must send a response back to the sender of the request.

```typescript
interface RequestMessage extends Message {

	/**
	 * The request id.
	 */
	id: number | string;

	/**
	 * The method to be invoked.
	 */
	method: string;

	/**
	 * The method's params.
	 */
	params?: any
}
```

#### Response Message

Response Message send as a result of a request. 

```typescript
interface ResponseMessage extends Message {
	/**
	 * The request id.
	 */
	id: number | string;

	/**
	 * The result of a request. This can be omitted in
	 * the case of an error.
	 */
	result?: any;

	/**
	 * The error object in case a request fails.
	 */
	error?: ResponseError<any>;
}

interface ResponseError<D> {
	/**
	 * A number indicating the error type that occured.
	 */
	code: number;

	/**
	 * A string providing a short decription of the error.
	 */
	message: string;

	/**
	 * A Primitive or Structured value that contains additional
	 * information about the error. Can be omitted.
	 */
	data?: D;
}

export namespace ErrorCodes {
	export const ParseError: number = -32700;
	export const InvalidRequest: number = -32600;
	export const MethodNotFound: number = -32601;
	export const InvalidParams: number = -32602;
	export const InternalError: number = -32603;
	export const serverErrorStart: number = -32099
	export const serverErrorEnd: number = -32000;
}
```
#### Notification Message

A notification message. A processed notification message must not send a response back. They work like events.

```typescript
interface NotificationMessage extends Message {
	/**
	 * The method to be invoked.
	 */
	method: string;

	/**
	 * The notification's params.
	 */
	params?: any
}
```

#### Cancellation Support

>**New:** The base protocol now offers support for request cancellation. To cancel a request a notification message with the following properties is sent:
 
_Notification_:
* method: '$/cancelRequest'
* params: `CancelParams` defined as follows:
```typescript
interface CancelParams {
	/**
	 * The request id to cancel.
	 */
	id: number | string;
}
```

A request that got canceled still needs to return from the server and send a response back. It can not be left open / hanging. This is in line with the JSON RPC protocol that requires that every request sends a reponse back. In addition it allows for returning partial results on cancel.

## Language Server Protocol

The language server protocol defines a set of JSON-RPC request, response and notification messages which are exchanged using the above base protocol. This section starts describing the basic JSON structures used in the protocol. The document uses TypeScript interfaces to describe these. Bases on the basic JSON structures the actual requests with their responses and the notifications are described.

### Basic JSON Structures

#### Position

Position in a text document expressed as zero-based line and character offset.

```typescript
interface Position {
	/**
	 * Line position in a document (zero-based).
	 */
	line: number;

	/**
	 * Character offset on a line in a document (zero-based).
	 */
	character: number;
}
```
#### Range

A range in a text document expressed as (zero-based) start and end positions.
```typescript
interface Range {
	/**
	 * The range's start position
	 */
	start: Position;

	/**
	 * The range's end position
	 */
	end: Position;
}
```

#### Location

Represents a location inside a resource, such as a line inside a text file.
```typescript
interface Location {
	uri: string;
	range: Range;
}
```

#### Diagnostic

Represents a diagnostic, such as a compiler error or warning. Diagnostic objects are only valid in the scope of a resource.

```typescript
interface Diagnostic {
	/**
	 * The range at which the message applies
	 */
	range: Range;

	/**
	 * The diagnostic's severity. Can be omitted. If omitted it is up to the
	 * client to interpret diagnostics as error, warning, info or hint.
	 */
	severity?: number;

	/**
	 * The diagnostic's code. Can be omitted.
	 */
	code?: number | string;

	/**
	 * A human-readable string describing the source of this
	 * diagnostic, e.g. 'typescript' or 'super lint'.
	 */
	source?: string;

	/**
	 * The diagnostic's message.
	 */
	message: string;
}
```

The protocol currently supports the following diagnostic severities:

```typescript
enum DiagnosticSeverity {
	/**
	 * Reports an error.
	 */
	Error = 1,
	/**
	 * Reports a warning.
	 */
	Warning = 2,
	/**
	 * Reports an information.
	 */
	Information = 3,
	/**
	 * Reports a hint.
	 */
	Hint = 4
}
```

#### Command

Represents a reference to a command. Provides a title which will be used to represent a command in the UI and, optionally, an array of arguments which will be passed to the command handler function when invoked.

```typescript
interface Command {
	/**
	 * Title of the command, like `save`.
	 */
	title: string;
	/**
	 * The identifier of the actual command handler.
	 */
	command: string;
	/**
	 * Arguments that the command handler should be
	 * invoked with.
	 */
	arguments?: any[];
}
```

#### TextEdit

A textual edit applicable to a text document.

```typescript
interface TextEdit {
	/**
	 * The range of the text document to be manipulated. To insert
	 * text into a document create a range where start === end.
	 */
	range: Range;

	/**
	 * The string to be inserted. For delete operations use an
	 * empty string.
	 */
	newText: string;
}
```

#### WorkspaceEdit

A workspace edit represents changes to many resources managed in the workspace.

 ```typescript
export interface WorkspaceEdit {
	/**
	 * Holds changes to existing resources.
	 */
	changes: { [uri: string]: TextEdit[]; };
}
```

#### TextDocumentIdentifier

Text documents are identified using an URI. On the protocol level URI's are passed as strings. The corresponding JSON structure looks like this:
```typescript
interface TextDocumentIdentifier {
	/**
	 * The text document's uri.
	 */
	uri: string;
}
```

#### TextDocumentItem

>**New:** an item to transfer a text document from the client to the server.

```typescript
export interface TextDocumentItem {
	/**
	 * The text document's uri.
	 */
	uri: string;

	/**
	 * The text document's language identifier
	 */
	languageId: string;

	/**
	 * The version number of this document (it will strictly increase after each
	 * change, including undo/redo).
	 */
	version: number;

	/**
	 * The content of the opened  text document.
	 */
	text: string;
}
```

#### VersionedTextDocumentIdentifier

>**New:** An identifier to denote a specific version of a text document.

```typescript
export interface VersionedTextDocumentIdentifier extends TextDocumentIdentifier {
	/**
	 * The version number of this document.
	 */
	version: number;
}
```

#### TextDocumentPositionParams

>**Changed**: was `TextDocumentPosition` in 1.0 with inlined parameters

A parameter literal used in requests to pass a text document and a position inside that document.

```typescript
export interface TextDocumentPositionParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The position inside the text document.
	 */
	position: Position;
}
```

### Actual Protocol

This section documents the actual language server protocol. It uses the following format:

* a header describing the request
* a _Request_ section describing the format of the request send. The method is a string identifying the request the params are documented using a TypeScript interface
* a _Response_ section describing the format of the response. The result item descibes the returned data in case of a success. The error.data describes the returned data in case of an error. Please remember that in case of a failure the response already contains an error.code and an error.message field. These fields are only speced if the protocol forces the use of certain error codes or messages. In cases where the server can decide on these values freely they arn't listed here.

#### Initialize Request

The initialize request is sent as the first request from the client to the server. 

_Request_
* method: 'initialize'
* params: `InitializeParams` defined as follows:
```typescript
interface InitializeParams {
	/**
	 * The process Id of the parent process that started
	 * the server.
	 */
	processId: number;

	/**
	 * The rootPath of the workspace. Is null
	 * if no folder is open.
	 */
	rootPath: string;

	/**
	 * The capabilities provided by the client (editor)
	 */
	capabilities: ClientCapabilities;
}
```
Where `ClientCapabilities` are currently empty:
```typescript
interface ClientCapabilities {
}
```

_Response_
* result: `InitializeResult` defined as follows:
```typescript
export InitializeResult {
	/**
	 * The capabilities the language server provides.
	 */
	capabilities: ServerCapabilities;
}
```
* error.data:
```typescript
interface InitializeError {
	/**
	 * Indicates whether the client should retry to send the
	 * initilize request after showing the message provided
	 * in the ResponseError.
	 */
	retry: boolean;
}
```
The server can signal the following capabilities:
```typescript
/**
 * Defines how the host (editor) should sync document changes to the language server.
 */
enum TextDocumentSyncKind {
	/**
	 * Documents should not be synced at all.
	 */
	None = 0,
	/**
	 * Documents are synced by always sending the full content of the document.
	 */
	Full = 1,
	/**
	 * Documents are synced by sending the full content on open. After that only incremental 
	 * updates to the document are send.
	 */
	Incremental = 2
}
```
```typescript
/**
 * Completion options.
 */
interface CompletionOptions {
	/**
	 * The server provides support to resolve additional information for a completion item.
	 */
	resolveProvider?: boolean;

	/**
	 * The characters that trigger completion automatically.
	 */
	triggerCharacters?: string[];
}
```
```typescript
/**
 * Signature help options.
 */
interface SignatureHelpOptions {
	/**
	 * The characters that trigger signature help automatically.
	 */
	triggerCharacters?: string[];
}
```
```typescript
/**
 * Code Lens options.
 */
interface CodeLensOptions {
	/**
	 * Code lens has a resolve provider as well.
	 */
	resolveProvider?: boolean;
}
```
```typescript
/**
 * Format document on type options
 */
interface DocumentOnTypeFormattingOptions {
	/**
	 * A character on which formatting should be triggered, like `}`.
	 */
	firstTriggerCharacter: string;
	/**
	 * More trigger characters.
	 */
	moreTriggerCharacter?: string[]
}
```
```typescript
interface ServerCapabilities {
	/**
	 * Defines how text documents are synced.
	 */
	textDocumentSync?: number;
	/**
	 * The server provides hover support.
	 */
	hoverProvider?: boolean;
	/**
	 * The server provides completion support.
	 */
	completionProvider?: CompletionOptions;
	/**
	 * The server provides signature help support.
	 */
	signatureHelpProvider?: SignatureHelpOptions;
	/**
	 * The server provides goto definition support.
	 */
	definitionProvider?: boolean;
	/**
	 * The server provides find references support.
	 */
	referencesProvider?: boolean;
	/**
	 * The server provides document highlight support.
	 */
	documentHighlightProvider?: boolean;
	/**
	 * The server provides document symbol support.
	 */
	documentSymbolProvider?: boolean;
	/**
	 * The server provides workspace symbol support.
	 */
	workspaceSymbolProvider?: boolean;
	/**
	 * The server provides code actions.
	 */
	codeActionProvider?: boolean;
	/**
	 * The server provides code lens.
	 */
	codeLensProvider?: CodeLensOptions;
	/**
	 * The server provides document formatting.
	 */
	documentFormattingProvider?: boolean;
	/**
	 * The server provides document range formatting.
	 */
	documentRangeFormattingProvider?: boolean;
	/**
	 * The server provides document formatting on typing.
	 */
	documentOnTypeFormattingProvider?: DocumentOnTypeFormattingOptions;
	/**
	 * The server provides rename support.
	 */
	renameProvider?: boolean
}
```

#### Shutdown Request

The shutdown request is sent from the client to the server. It asks the server to shutdown, but to not exit (otherwise the response might not be delivered correctly to the client). There is a separate exit notification that asks the server to exit.

_Request_
* method: 'shutdown'
* params: undefined

_Response_
* result: undefined
* error: code and message set in case an exception happens during shutdown request.

#### Exit Notification

A notification to ask the server to exit its process.

_Notification_
* method: 'exit'
* params: undefined

#### ShowMessage Notification

The show message notification is sent from a server to a client to ask the client to display a particular message in the user interface.

_Notification_:
* method: 'window/showMessage'
* params: `ShowMessageParams` defined as follows:
```typescript
interface ShowMessageParams {
	/**
	 * The message type. See {@link MessageType}
	 */
	type: number;

	/**
	 * The actual message
	 */
	message: string;
}
```
Where the type is defined as follows:
```typescript
enum MessageType {
	/**
	 * An error message.
	 */
	Error = 1,
	/**
	 * A warning message.
	 */
	Warning = 2,
	/**
	 * An information message.
	 */
	Info = 3,
	/**
	 * A log message.
	 */
	Log = 4
}
```

#### ShowMessage Request

>**New:** he show message request is sent from a server to a client to ask the client to display a particular message in the user interface. In addition to the show message notification the request allows to pass actions and to wait for an answer from the client.

_Notification_:
* method: 'window/showMessageRequest'
* params: `ShowMessageRequestParams` defined as follows:
```typescript
export interface ShowMessageRequestParams {
	/**
	 * The message type. See {@link MessageType}
	 */
	type: number;

	/**
	 * The actual message
	 */
	message: string;

	/**
	 * The message action items to present.
	 */
	actions?: MessageActionItem[];
}
```
Where the `MessageActionItem` is defined as follows:
```typescript
export interface MessageActionItem {
	/**
	 * A short title like 'Retry', 'Open Log' etc.
	 */
	title: string;
}
```

#### LogMessage Notification

The log message notification is send from the server to the client to ask the client to log a particular message.

_Notification_:
* method: 'window/logMessage'
* params: `LogMessageParams` defined as follows:
```typescript
interface LogMessageParams {
	/**
	 * The message type. See {@link MessageType}
	 */
	type: number;

	/**
	 * The actual message
	 */
	message: string;
}
```
Where type is defined as above.

#### Telemetry Notification

>**New:** The telemetry notification is send from the server to the client to ask the client to log a telemetry event.

_Notification_:
* method: 'telemetry/event'
* params: 'any'

#### DidChangeConfiguration Notification

A notification send from the client to the server to signal the change of configuration settings.

_Notification_:
* method: 'workspace/didChangeConfiguration',
* params: `DidChangeConfigurationParams` defined as follows:
```typescript
interface DidChangeConfigurationParams {
	/**
	 * The actual changed settings
	 */
	settings: any;
}
```

#### DidOpenTextDocument Notification

The document open notification is sent from the client to the server to signal newly opened text documents. The document's truth is now managed by the client and the server must not try to read the document's truth using the document's uri.

_Notification_:
* method: 'textDocument/didOpen'
* params: `DidOpenTextDocumentParams` defined as follows:
```typescript
interface DidOpenTextDocumentParams extends TextDocumentIdentifier {
	/**
	 * The document that was opened.
	 */
	textDocument: TextDocumentItem;
}
```

#### DidChangeTextDocument Notification

>**Changed**: The document change notification is sent from the client to the server to signal changes to a text document. In 2.0 the shape of the params has changed to include proper version numbers and language ids.

_Notification_:
* method: 'textDocument/didChange'
* params: `DidChangeTextDocumentParams` defined as follows:
```typescript
export interface DidChangeTextDocumentParams {
	/**
	 * The document that did change. The version number points
	 * to the version after all provided content changes have
	 * been applied.
	 */
	textDocument: VersionedTextDocumentIdentifier;

	/**
	 * The actual content changes.
	 */
	contentChanges: TextDocumentContentChangeEvent[];
}
```
```typescript
/**
 * An event describing a change to a text document. If range and rangeLength are omitted
 * the new text is considered to be the full content of the document.
 */
interface TextDocumentContentChangeEvent {
	/**
	 * The range of the document that changed.
	 */
	range?: Range;

	/**
	 * The length of the range that got replaced.
	 */
	rangeLength?: number;

	/**
	 * The new text of the document.
	 */
	text: string;
}
```

#### DidCloseTextDocument Notification

The document close notification is sent from the client to the server when the document got closed in the client. The document's truth now exists where the document's uri points to (e.g. if the document's uri is a file uri the truth now exists on disk).

>**Changed**: in 2.0 the params are of type `DidCloseTextDocumentParams` which contains a reference to a text document.
_Notification_:
* method: 'textDocument/didClose'
* param: `DidCloseTextDocumentParams` defined as follows:
```typescript
export interface DidCloseTextDocumentParams {
	/**
	 * The document that was closed.
	 */
	textDocument: TextDocumentIdentifier;
}
```

#### DidSaveTextDocument Notification

>**New:** The document save notification is sent from the client to the server when the document for saved in the clinet.

* method: 'textDocument/didSave'
* param: `DidSaveTextDocumentParams` defined as follows:
```typescript
export interface DidSaveTextDocumentParams {
	/**
	 * The document that was closed.
	 */
	textDocument: TextDocumentIdentifier;
}
```

#### DidChangeWatchedFiles Notification

The watched files notification is sent from the client to the server when the client detects changes to file watched by the lanaguage client.

_Notification_:
* method: 'workspace/didChangeWatchedFiles'
* params: `DidChangeWatchedFilesParams` defined as follows:
```typescript
interface DidChangeWatchedFilesParams {
	/**
	 * The actual file events.
	 */
	changes: FileEvent[];
}
```
Where FileEvents are described as follows:
```typescript
/**
 * The file event type
 */
enum FileChangeType {
	/**
	 * The file got created.
	 */
	Created = 1,
	/**
	 * The file got changed.
	 */
	Changed = 2,
	/**
	 * The file got deleted.
	 */
	Deleted = 3
}
```
```typescript
/**
 * An event describing a file change.
 */
interface FileEvent {
	/**
	 * The file's uri.
	 */
	uri: string;
	/**
	 * The change type.
	 */
	type: number;
}
```

#### PublishDiagnostics Notification

Diagnostics notification are sent from the server to the client to signal results of validation runs.

_Notification_
* method: 'textDocument/publishDiagnostics'
* params: `PublishDiagnosticsParams` defined as follows:
```typescript
interface PublishDiagnosticsParams {
	/**
	 * The URI for which diagnostic information is reported.
	 */
	uri: string;

	/**
	 * An array of diagnostic information items.
	 */
	diagnostics: Diagnostic[];
}
```

#### Completion Request

The Completion request is sent from the client to the server to compute completion items at a given cursor position. Completion items are presented in the [IntelliSense](https://code.visualstudio.com/docs/editor/editingevolved#_intellisense) user interface. If computing complete completion items is expensive servers can additional provide a handler for the resolve completion item request. This request is send when a completion item is selected in the user interface.

>**Changed:**: in 2.0 the requests uses `TextDocumentPositionParams` with a proper `textDocument` and `position` property. In 1.0 the uri of the referenced text document was inlined into the params object. 

_Request_
* method: 'textDocument/completion'
* params: [`TextDocumentPositionParams`](#textdocumentposition)

_Response_
* result: `CompletionItem[] | CompletionList`
```typescript
/**
 * Represents a collection of [completion items](#CompletionItem) to be presented
 * in the editor.
 */
interface CompletionList {
	/**
	 * This list it not complete. Further typing should result in recomputing
	 * this list.
	 */
	isIncomplete: boolean;
	/**
	 * The completion items.
	 */
	items: CompletionItem[];
}
```
```typescript
interface CompletionItem {
	/**
	 * The label of this completion item. By default
	 * also the text that is inserted when selecting
	 * this completion.
	 */
	label: string;
	/**
	 * The kind of this completion item. Based of the kind
	 * an icon is chosen by the editor.
	 */
	kind?: number;
	/**
	 * A human-readable string with additional information
	 * about this item, like type or symbol information.
	 */
	detail?: string;
	/**
	 * A human-readable string that represents a doc-comment.
	 */
	documentation?: string;
	/**
	 * A string that shoud be used when comparing this item
	 * with other items. When `falsy` the label is used.
	 */
	sortText?: string;
	/**
	 * A string that should be used when filtering a set of
	 * completion items. When `falsy` the label is used.
	 */
	filterText?: string;
	/**
	 * A string that should be inserted a document when selecting
	 * this completion. When `falsy` the label is used.
	 */
	insertText?: string;
	/**
	 * An edit which is applied to a document when selecting
	 * this completion. When an edit is provided the value of
	 * insertText is ignored.
	 */
	textEdit?: TextEdit;
	/**
	 * An data entry field that is preserved on a completion item between
	 * a completion and a completion resolve request.
	 */
	data?: any
}
```
Where `CompletionItemKind` is defined as follows:
```typescript
/**
 * The kind of a completion entry.
 */
enum CompletionItemKind {
	Text = 1,
	Method = 2,
	Function = 3,
	Constructor = 4,
	Field = 5,
	Variable = 6,
	Class = 7,
	Interface = 8,
	Module = 9,
	Property = 10,
	Unit = 11,
	Value = 12,
	Enum = 13,
	Keyword = 14,
	Snippet = 15,
	Color = 16,
	File = 17,
	Reference = 18
}
```
* error: code and message set in case an exception happens during the completion request.

#### Completion Item Resolve Request

The request is sent from the client to the server to resolve additional information for a given completion item.

_Request_
* method: 'completionItem/resolve'
* param: `CompletionItem`

_Response_
* result: `CompletionItem`
* error: code and message set in case an exception happens during the completion resolve request.

#### Hover

The hover request is sent from the client to the server to request hover information at a given text document position.

>**Changed:**: in 2.0 the requests uses `TextDocumentPositionParams` with a proper `textDocument` and `position` property. In 1.0 the uri of the referenced text document was inlined into the params object. 

_Request_
* method: 'textDocument/hover'
* param: [`TextDocumentPositionParams`](#textdocumentpositionparams)

_Response_
* result: `Hover` defined as follows:
```typescript
/**
 * The result of a hove request.
 */
interface Hover {
	/**
	 * The hover's content
	 */
	contents: MarkedString | MarkedString[];
	
	/**
	 * An optional range
	 */
	range?: Range;
}
```
Where `MarkedString` is defined as follows:
```typescript
type MarkedString = string | { language: string; value: string };
```
* error: code and message set in case an exception happens during the hover request.

#### Signature Help

The signature help request is sent from the client to the server to request signature information at a given cursor position.

>**Changed:**: in 2.0 the requests uses `TextDocumentPositionParams` with a proper `textDocument` and `position` property. In 1.0 the uri of the referenced text document was inlined into the params object. 

_Request_
* method: 'textDocument/signatureHelp'
* param: [`TextDocumentPositionParams`](#textdocumentpositionparams)

_Response_
* result: `SignatureHelp` defined as follows:
```typescript
/**
 * Signature help represents the signature of something
 * callable. There can be multiple signature but only one
 * active and only one active parameter.
 */
interface SignatureHelp {
	/**
	 * One or more signatures.
	 */
	signatures: SignatureInformation[];
	
	/**
	 * The active signature.
	 */
	activeSignature?: number;
	
	/**
	 * The active parameter of the active signature.
	 */
	activeParameter?: number;
}
```
```typescript
/**
 * Represents the signature of something callable. A signature
 * can have a label, like a function-name, a doc-comment, and
 * a set of parameters.
 */
interface SignatureInformation {
	/**
	 * The label of this signature. Will be shown in
	 * the UI.
	 */
	label: string;
	
	/**
	 * The human-readable doc-comment of this signature. Will be shown
	 * in the UI but can be omitted.
	 */
	documentation?: string;
	
	/**
	 * The parameters of this signature.
	 */
	parameters?: ParameterInformation[];
}
```
```typescript
/**
 * Represents a parameter of a callable-signature. A parameter can
 * have a label and a doc-comment.
 */
interface ParameterInformation {
	/**
	 * The label of this signature. Will be shown in
	 * the UI.
	 */
	label: string;
	
	/**
	 * The human-readable doc-comment of this signature. Will be shown
	 * in the UI but can be omitted.
	 */
	documentation?: string;
}
```
* error: code and message set in case an exception happens during the signature help request.

#### Goto Definition

The goto definition request is sent from the client to the server to to resolve the defintion location of a symbol at a given text document position.

>**Changed:**: in 2.0 the requests uses `TextDocumentPositionParams` with a proper `textDocument` and `position` property. In 1.0 the uri of the referenced text document was inlined into the params object. 

_Request_
* method: 'textDocument/definition'
* param: [`TextDocumentPositionParams`](#textdocumentpositionparams)

_Response_:
* result: [`Location`](#location) | [`Location`](#location)[]
* error: code and message set in case an exception happens during the definition request.

#### Find References

The references request is sent from the client to the server to resolve project-wide references for the symbol denoted by the given text document position.

>**Changed:**: in 2.0 the requests uses `TextDocumentPositionParams` with a proper `textDocument` and `position` property. In 1.0 the uri of the referenced text document was inlined into the params object. 

_Request_
* method: 'textDocument/references'
* param: `ReferenceParams` defined as follows:
```typescript
interface ReferenceParams extends TextDocumentPositionParams {
	context: ReferenceContext
}
```
```typescript
interface ReferenceContext {
	/**
	 * Include the declaration of the current symbol.
	 */
	includeDeclaration: boolean;
}
```
* error: code and message set in case an exception happens during the reference request.

#### Document Highlights

The document highlight request is sent from the client to the server to to resolve a document highlights for a given text document position.

>**Changed:**: in 2.0 the requests uses `TextDocumentPositionParams` with a proper `textDocument` and `position` property. In 1.0 the uri of the referenced text document was inlined into the params object. 

_Request_
* method: 'textDocument/documentHighlight'
* param: [`TextDocumentPositionParams`](#textdocumentpositionparams)

_Response_
* result: `DocumentHighlight` defined as follows:
```typescript
/**
 * A document highlight is a range inside a text document which deserves
 * special attention. Usually a document highlight is visualized by changing
 * the background color of its range.
 */
interface DocumentHighlight {
	/**
	 * The range this highlight applies to.
	 */
	range: Range;

	/**
	 * The highlight kind, default is DocumentHighlightKind.Text.
	 */
	kind?: number;
}
```
```typescript
/**
 * A document highlight kind.
 */
enum DocumentHighlightKind {
	/**
	 * A textual occurrance.
	 */
	Text = 1,

	/**
	 * Read-access of a symbol, like reading a variable.
	 */
	Read = 2,

	/**
	 * Write-access of a symbol, like writing to a variable.
	 */
	Write = 3
}
```
* error: code and message set in case an exception happens during the document highlight request.


#### Document Symbols

The document symbol request is sent from the client to the server to list all symbols found in a given text document.

>**Changed:** in 2.0 the request uses `DocumentSymbolParams` instead of a single uri.

_Request_
* method: 'textDocument/documentSymbol'
* param: `DocumentSymbolParams` defined as follows:
```typescript
export interface DocumentSymbolParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;
}
```

_Response_
* result: `SymbolInformation`[] defined as follows:
```typescript
/**
 * Represents information about programming constructs like variables, classes,
 * interfaces etc.
 */
interface SymbolInformation {
	/**
	 * The name of this symbol.
	 */
	name: string;

	/**
	 * The kind of this symbol.
	 */
	kind: number;

	/**
	 * The location of this symbol.
	 */
	location: Location;

	/**
	 * The name of the symbol containing this symbol.
	 */
	containerName?: string;
}
```
Where the `kind` is defined like this:
```typescript
/**
 * A symbol kind.
 */
export enum SymbolKind {
	File = 1,
	Module = 2,
	Namespace = 3,
	Package = 4,
	Class = 5,
	Method = 6,
	Property = 7,
	Field = 8,
	Constructor = 9,
	Enum = 10,
	Interface = 11,
	Function = 12,
	Variable = 13,
	Constant = 14,
	String = 15,
	Number = 16,
	Boolean = 17,
	Array = 18,
}
```
* error: code and message set in case an exception happens during the document symbol request.

#### Workspace Symbols

The workspace symbol request is sent from the client to the server to list project-wide symbols matching the query string.

_Request_
* method: 'workspace/symbol'
* param: `WorkspaceSymbolParams` defined as follows:
```typescript
/**
 * The parameters of a Workspace Symbol Request.
 */
interface WorkspaceSymbolParams {
	/**
	 * A non-empty query string
	 */
	query: string;
}
```

_Response_
* result: `SymbolInformation[]` as defined above.
* error: code and message set in case an exception happens during the workspace symbol request.

#### Code Action

The code action request is sent from the client to the server to compute commands for a given text document and range. The request is trigger when the user moves the cursor into an problem marker in the editor or presses the lightbulb associated with a marker.

_Request_
* method: 'textDocument/codeAction'
* param: `CodeActionParams` defined as follows:
```typescript
/**
 * Params for the CodeActionRequest
 */
interface CodeActionParams {
	/**
	 * The document in which the command was invoked.
	 */
	textDocument: TextDocumentIdentifier;
	
	/**
	 * The range for which the command was invoked.
	 */
	range: Range;
	
	/**
	 * Context carrying additional information.
	 */
	context: CodeActionContext;
}
```
```typescript
/**
 * Contains additional diagnostic information about the context in which
 * a code action is run.
 */
interface CodeActionContext {
	/**
	 * An array of diagnostics.
	 */
	diagnostics: Diagnostic[];
}
```

_Response_
* result: [`Command[]`](#command) defined as follows:
* error: code and message set in case an exception happens during the code action request.

#### Code Lens

The code lens request is sent from the client to the server to compute code lenses for a given text document.

>**Changed:** in 2.0 the request uses `CodeLensParams` instead of a single uri.

_Request_
* method: 'textDocument/codeLens'
* param: `CodeLensParams` defined as follows:
```typescript
export interface CodeLensParams {
	/**
	 * The document to request code lens for.
	 */
	textDocument: TextDocumentIdentifier;
}
```

_Response_
* result: `CodeLens[]` defined as follows:
```typescript
/**
 * A code lens represents a command that should be shown along with
 * source text, like the number of references, a way to run tests, etc.
 *
 * A code lens is _unresolved_ when no command is associated to it. For performance
 * reasons the creation of a code lens and resolving should be done to two stages.
 */
export interface CodeLens {
	/**
	 * The range in which this code lens is valid. Should only span a single line.
	 */
	range: Range;

	/**
	 * The command this code lens represents.
	 */
	command?: Command;
	
	/**
	 * An data entry field that is preserved on a code lens item between
	 * a code lens and a code lens resolve request.
	 */
	data?: any
}
```
* error: code and message set in case an exception happens during the code lens request.

#### Code Lens Resolve

The code lens resolve request is sent from the clien to the server to resolve the command for a given code lens item.

_Request_
* method: 'codeLens/resolve'
* param: `CodeLens`

_Response_
* result: `CodeLens`
* error: code and message set in case an exception happens during the code lens resolve request.

#### Document Formatting

The document formatting resquest is sent from the server to the client to format a whole document.

_Request_
* method: 'textDocument/formatting'
* param: `DocumentFormattingParams` defined as follows
```typescript
interface DocumentFormattingParams {
	/**
	 * The document to format.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The format options
	 */
	options: FormattingOptions;
}
```
```typescript
/**
 * Value-object describing what options formatting should use.
 */
interface FormattingOptions {
	/**
	 * Size of a tab in spaces.
	 */
	tabSize: number;

	/**
	 * Prefer spaces over tabs.
	 */
	insertSpaces: boolean;

	/**
	 * Signature for further properties.
	 */
	[key: string]: boolean | number | string;
}
```

_Response_
* result: [`TextEdit[]`](#textedit) describing the modification to the document to be formatted.
* error: code and message set in case an exception happens during the formatting request.

#### Document Range Formatting

The document range formatting request is sent from the client to the server to format a given range in a document.

_Request_
* method: 'textDocument/rangeFormatting',
* param: `DocumentRangeFormattingParams` defined as follows
```typescript
interface DocumentRangeFormattingParams {
	/**
	 * The document to format.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The range to format
	 */
	range: Range;

	/**
	 * The format options
	 */
	options: FormattingOptions;
}
```

_Response_
* result: [`TextEdit[]`](#textedit) describing the modification to the document to be formatted.
* error: code and message set in case an exception happens during the range formatting request.

#### Document on Type Formatting

The document on type formatting request is sent from the client to the server to format parts of the document during typing.

_Request_
* method: 'textDocument/onTypeFormatting'
* param: `DocumentOnTypeFormattingParams` defined as follows
```typescript
interface DocumentOnTypeFormattingParams {
	/**
	 * The document to format.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The position at which this request was send.
	 */
	position: Position;

	/**
	 * The character that has been typed.
	 */
	ch: string;
	
	/**
	 * The format options.
	 */
	options: FormattingOptions;
}
```

_Response_
* result: [`TextEdit[]`](#textedit) describing the modification to the document.
* error: code and message set in case an exception happens during the range formatting request.

#### Rename

The rename request is sent from the client to the server to do a workspace wide rename of a symbol.

_Request_
* method: 'textDocument/rename'
* param: `RenameParams` defined as follows
```typescript
export interface RenameParams {
	/**
	 * The document to format.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The position at which this request was send.
	 */
	position: Position;

	/**
	 * The new name of the symbol. If the given name is not valid the
	 * request must return a [ResponseError](#ResponseError) with an
	 * appropriate message set.
	 */
	newName: string;
}
```

_Response_
* result: [`WorkspaceEdit`](#workspaceedit) describing the modification to the workspace.
* error: code and message set in case an exception happens during the rename request.
