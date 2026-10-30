# AI BPMN Diagram Analysis & Validation Engine

Silent incidents, caught before deploy.

A Chrome extension for the Camunda 8 Web Modeler that checks your BPMN diagram while you are still building it, not after Camunda Operate has already logged the failure.

Built by ilume Informatik AG, an IT consulting company specializing in process automation (Camunda, UiPath, RPA, ServiceNow, IBM BAW, n8n).

The problem

A diagram can pass every syntax check and still fail silently in production. The usual causes are small and easy to miss while modeling:

a field name misspelled by one letter, read correctly in one place and incorrectly in another
a value read before it was ever written on the path that leads to it
a FEEL expression that evaluates to null at runtime, with no incident raised
a credential hardcoded into the diagram instead of stored in a vault

None of this throws an error at deploy time. It simply runs, until somewhere downstream the process stalls, or a secret ends up somewhere it never should have.

What it does

The extension runs a static analysis of the open BPMN diagram directly inside the Camunda 8 Web Modeler:

Wrong-path and read-before-write detection. Finds FEEL expressions that reference a field incorrectly, or read a value before it is set on any path that reaches that point.
Diagram-linked findings. Every finding is a card in a side panel, tied to the exact element on the canvas. Open a card and the related element highlights; click the element and the matching card opens.
Hardcoded credential detection. Flags values matching known credential patterns (such as AWS access keys) or common credential field names as a separate finding, before they leave the Modeler.
Second opinion via Claude on AWS Bedrock. For findings that need more than pattern matching, "How to fix" sends what the static analysis already found to Claude, which returns a verdict, the reasoning behind it, and a concrete fix. The model is given one finding at a time and is not allowed to invent new ones; it can also return "false alarm" if the finding does not hold up.
How it works

The extension is a Manifest V3 Chrome extension. A content script runs inside the Camunda 8 Web Modeler and reads the open diagram through the Modeler's internal API. Findings are computed client-side and rendered as overlay markers on the diagram and as cards in a side panel.

The Bedrock call is opt-in and happens per finding, not automatically for the whole diagram:

The side panel sends one finding to a small AWS Lambda function.
The Lambda function holds the only AWS credentials involved. It calls Bedrock and returns the result.
A shared token between the extension and the Lambda function is the only thing that gates the call; the extension itself never holds AWS credentials.

Nothing is sent to Bedrock unless a card's "How to fix" button is pressed.

Installation
Clone this repository.
Open chrome://extensions, enable Developer Mode, and choose "Load unpacked".
Select the repository folder.
Open your Camunda 8 Web Modeler and open a diagram. The extension icon appears in the Chrome toolbar next to your other extensions.
Click the icon to open the side panel, then "Analyze diagram".

The static analysis works without any further setup. The AI second-opinion stage requires the AWS-side configuration below.

Configuration
Modeler host

Add your Camunda 8 Web Modeler host to host_permissions in manifest.json.

AI second-opinion stage (optional)
Deploy lambda_function.py as an AWS Lambda function (Python 3.12) with bedrock:InvokeModel permission, scoped to the Bedrock model you intend to use.
Set a shared secret as the Lambda's GUARD_SHARED_TOKEN environment variable, and enable a Function URL for it.
Add the Function URL to host_permissions in manifest.json.
Set LLM_CONFIG.endpoint and LLM_CONFIG.token in llm.js to match.

Without this step, the extension still runs the full static analysis; only the "How to fix" second opinion stays unavailable.

Status

This is a proof of concept, built by ilume Informatik AG to demonstrate what is possible with Camunda 8, Chrome, and an AI model wired in for a second opinion. It is not a maintained, supported product, and comes with no warranty or service level agreement. If you are interested in a version tailored to your own environment, get in touch.

Tech stack
Camunda 8 Web Modeler
Chrome extension, Manifest V3
AWS Lambda
Claude on AWS Bedrock
