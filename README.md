# Cloud Browser

## What is Cloud Browser?

It is a programmable, read-only browser based interface to your AWS resources. 

## Why do we need Cloud Browser when the AWS console exists?

1. The AWS console is a general purpose tool. That being the case, it is very common to have to click through a myriad menu options to surface information relevant to your specific use case.

2. There is a lot of (understandable) handwringing that happens around giving AWS access to contractors and consultants. Adding and updating IAM policies to make sure every consultant/contractor is only seeing what they are supposed to see is an exercise no one enjoys doing. Plus, this busywork takes time away from bctually uilding stuff.

3. Product and engineering leadership want access to AWS without the cruft and want an easy way to get things done in there without (a) having to learn the ins and outs of AWS (b) having to wait for days for a principal engineer to actually build what they want. An example of this could be a SVP of technology wanting to show some interesting access patterns to the SVP of marketing using CloudFront logs.

## How does Cloud Browser work?

To answer this, we need to define two roles: a Cloud Browser Guru and a User. 

A *Guru* is the person responsible for maintaining and enhancing the Cloud Browser interface for your AWS account. The Guru is functionally equivalent to a Jira admin. They can edit and enhance the interface that a User sees. 

A *User* accesses Cloud Browser on their browser and uses it to read/query data from AWS. A User has no role to play to make Cloud Browser usable.

To help a *Guru* build a use case specific AWS interface, Cloud Browser has defined a DSL.

The key insight behind the DSL is that a typical AWS case involves _listing_ resources, following by _drilling_ into one instance of that resource, then _seeing_ more details about that resource. For example, you may list all your Lambdas, then drill into one Lambda, then check its concurrency setting or handler code.

Luckily for us, AWS provides a Javascript SDK to be used in the browser to help us *list*, *drill*, and *show* specific resources.

Here is an example of this in action:

```
{
	"Lambda": {
		"list": {
			"func": "listFunctions",
			"key": "Functions",
			"section_header": "Your Lambda Functions",
			"title": "FunctionName",
			"subtitle": "Version, Description, Runtime",
			"next": "drill",
			"loading_message": "Downloading Lambdas..."
		},
		"drill": {
			"func": "getFunction",
			"params": "FunctionName=$FunctionName",
			"section_header": "Your Function",
			"title": "Configuration.FunctionName",
			"subtitle": "Configuration.Runtime, Configuration.CodeSize, Configuration.LastModified, Code.Location",
			"loading_message": "Getting Lambda details...",
			"empty_message": "No function found",
			"formats": {
				"subtitle": {
					"Configuration.LastModified": stringAsDate,
					"Code.Location": stringAsLink,
					"withLabels": true
				}
			},
			"next": "show",
			"action_hooks": {
				"openai": {
					"prompt": "explain this code in plain english to a senior software engineer:\n ",
					"parameter": "Code.Location",
					"injection_method": "download",
					"button_label": "Explain Function",
					"textarea_default_value": "\nPaste this Lambda function's code here then have OpenAI explain what it does..."
				}
			}
		},
		"show": {
			"func": "modal",
			"title": "Configuration.FunctionName",
			"section_header": "Your function details",
			"subtitle": "*",
			"empty_message": "No details found",
			"formats": {
				"subtitle": {
					"Configuration.Layers": jsonAsString,
					"withLabels": true
				}
			},
			"next": "show",
			"action_hooks": {
				"openai": {
					"prompt": "explain this code in plain english to a senior software engineer:\n ",
					"parameter": "Code.Location",
					"injection_method": "download",
					"button_label": "Explain Function"
				}
			}
		},
		"apiVersion": "2015 - 03 - 31"
	},
}
```

`Lambda` corresponds to the Lambda client object defined by the SDK.
`list`, `drill`, and `show` are three keys inside Lambda.
`func`: is the name of the function that the you would like to call on the SDK's Lambda object. E.g., `listFunctions` corresponds to [this](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Lambda.html#listFunctions-property) Lambda API. `func` has a special magic value of `modal` which displays data in a `modal`
`section_header`: is the title of the page as shown in the browser to the user. 
`key`: sometimes, the AWS CLI returns a list of data attached to a specific key. E.g., the response to `listFunctions` results in a dictionary with root `Functions`. Specifying the `key` as `Functions` makes it easier to identify the array whose elements we need to show as a list.
`title`: the name of the resource. E.g., treating `FunctionName` as the title lets you show the name of the Lambda function
`subtitle`: some extra information about the resource to help users correctly identify the resource as the title lets you show the name of the Lambda function. The `subtitle` can be one of three values: '\*', \'\', or a comma delimited string. In this case, `Version, Description, Runtime` are a comma delimited set of keys each of which include some valuable information about the Lambda.
`next`: is the name of the next function to call when the user clicks on a specific lambda. Since `list` is followed by `drill`-ing into a specific function, `next` is set to `drill`.
`loading_message` is the message to be shown while the underlying data is being requested from AWS by SDK 
`empty_message`: is the message to be shown when there is no data available to be shown.
`formats.title` | `formats.subtitle`: how specific fields within the title or subtitle should be formatted. E.g., if your subtitle includes a date, you could request it to be formatted into a human friendly form. Links can be formatted in anchor tags with the anchor text of *Link*.
`withLabels`: specifies that the formatting should include the label from where the data is being read. E.g., if `withLabels` is set to true, subtitle will say `Version=$Latest, Runtime=nodejs14.x,...` and so on.
`action_hooks`: this is a bit of a value added feature. You could ask the interface to display various bells and whistles at any level of the interface. E.g., you could ask the interface to display an OpenAI textbox and set up a prompt to tell OpenAI what to do. In this example, we have asked OpenAI to explain what a given Lambda function does.
