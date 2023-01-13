# Getting Started

1. Clone this repo then `cd` into the cloned directory
2. Run `http-server` to serve the index file 
3. Visit the Cloud Browser in your browser

At this point, you should see the `Settings` page where you can enter your `AWS Key` and `Secret`. Save those values locally to your browser's Local Storage.

## Prerequisites
1. Make sure you have a tool like `http-server` installed to serve static pages locally
2. AWS account with Key and Secret. The default region is set to `us-east-1`. Feel free to edit the value in index.html if you want to use some other area.

# What is Cloud Browser?

It is a programmable, read-only browser based interface to your AWS resources. 

# What technologies does it use?

1. AWS SDK aws-sdk-2.941.0 served from https://sdk.amazonaws.com/js/aws-sdk-2.941.0.min.js. I used V2 of AWS CLI to keep the setup simple. You can reduce the amount of code being sent over the wire by switching to V3 and only include AWK SDK objects that you care about.
2. Bulma CSS 0.9.3 served from https://cdn.jsdelivr.net/npm/bulma@0.9.3/css/bulma.min.css. it's the only CSS I know but I'm sure a half competent UI developer can make the interface look very nice
3. VueJS 3 from https://cdn.jsdelivr.net/npm/vue/dist/vue.js. I use it in the browser mode to keep the setup simple.
4. FontAwesome
5. GoatCounter for metrics

# How safe/secure is Cloud Browser?

Cloud Browser runs purely inside your browser and makes no server/backend network calls to anything other than AWS and OpenAI. You can verify this by monitoring your network traffic in browser DevTools. So, the app is as secure as your browser.

# Why do we need Cloud Browser when the AWS console exists?

1. The AWS console is a general purpose tool. That being the case, it is very common to have to click through a myriad menu options to surface information relevant to your specific use case.

2. There is a lot of (understandable) handwringing that happens around giving AWS access to contractors and consultants. Adding and updating IAM policies to make sure every consultant/contractor is only seeing what they are supposed to see is an exercise no one enjoys doing. Plus, this busywork takes time away from actually building stuff.

3. Product and engineering leadership want access to AWS without the cruft and want an easy way to get things done in there without (a) having to learn the ins and outs of AWS (b) having to wait for days for a principal engineer to actually build what they want. An example of this could be a SVP of technology wanting to show some interesting access patterns to the SVP of marketing using CloudFront logs.

# How does Cloud Browser work?

To answer this, we need to define two roles: a Cloud Browser Guru and a User. 

A *Guru* is the person responsible for maintaining and enhancing the Cloud Browser interface for your AWS account. The Guru is functionally equivalent to a Jira admin. They can edit and enhance the interface that a User sees. 

A *User* accesses Cloud Browser on their browser and uses it to read/query data from AWS. A User has no role to play to make Cloud Browser usable.

To help a *Guru* build a use case specific AWS interface, Cloud Browser has defined a DSL.

The key insight behind the DSL is that a typical AWS case involves _listing_ resources, following by _drilling_ into one instance of that resource, then _seeing_ more details about that resource. For example, you may list all your Lambdas, then drill into one Lambda, then check its concurrency setting or handler code.

Luckily for us, AWS provides a Javascript SDK to be used in the browser to help us *list*, *drill*, and *show* specific resources.

Here is an example of this in action. Read the Glossary further down in this document while referencing this example.

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
					"button_label": "Explain Function"
				}
			}
		},
		"apiVersion": "2015 - 03 - 31"
	},
}
```

# Glossary

`Lambda` corresponds to the Lambda client object defined by the SDK.

`list`, `drill`, and `show` are three keys inside Lambda. `list` is the only required key. Your next step could be called anything you want so long as you specify the correct `next` value defined below.

`func`: is the name of the function that the you would like to call on the SDK's Lambda object. E.g., `listFunctions` corresponds to [this](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Lambda.html#listFunctions-property) Lambda API. `func` can have a special magic value of `modal` which displays data in a browser modal

`params`: a comma separated `Name1=$Value1, Name2=$Value2` string. Here, `Name1 | Name2 | ...` correspond to the param name expected by the AWS SDK. `$Value1 | $Value2 | ...` correspond to the value of the variable whose name is `Value1 | Value2 | ...`. E.g. if the params string is `foo=$bar`, then the API call would expect a variable called `foo` and the value of `foo` would be value of `bar`. 

`section_header`: is the title of the page as shown in the browser to the user. 

`key`: sometimes, AWS returns a list of data attached to a specific key. E.g., the response to `listFunctions` results in a dictionary with root `Functions`. Specifying the `key` as `Functions` makes it easy to identify the array whose elements we need to show as a list.

`title`: the name of the resource. E.g., setting title to `FunctionName` lets you display the name of the Lambda function.

`subtitle`: some extra information about the resource to help users correctly identify the resource. The `subtitle` can be one of three values: '\*', \'\', or a comma delimited string. In this case, `Version, Description, Runtime` are a comma delimited set of keys. Their values, concatenated as a comma delimited string, form the displayed subtitle.

`next`: is the name of the next function to call when the user clicks on a specific lambda. Since `list` is followed by `drill`-ing into a specific function, `next` is set to `drill`.

`loading_message` is the message to be shown while the underlying data is being requested from AWS by SDK. 

`empty_message`: is the message to be shown when there is no data available to be shown.

`formats.title` | `formats.subtitle`: how specific fields within the title or subtitle should be formatted. E.g., if your subtitle includes a date, you could request it to be formatted into a human friendly form. Links can be formatted as anchor tags with the anchor text of *Link*.

`withLabels`: specifies that the formatting should include the label from where the data is being read. E.g., if `withLabels` is set to true, subtitle will say `Version=$Latest, Runtime=nodejs14.x,...` and so on. If `withLabel` is not defined or is set to false, the subtitle will say `$Latest, nodejs14.x, ...` without the labels.

`action_hooks`: this is a bit of a value added feature. You could ask the interface to display various bells and whistles at any level of the interface. E.g., you could ask the interface to display an OpenAI textbox and set up a prompt to tell OpenAI what to do. In this example, we have asked OpenAI to explain what a given Lambda function does. 

A special note about the `prompt` field inside `action_hooks`. Since the `prompt` might only make sense if some contextual information from the AWS resource is included, the `prompt` can include two special string `$title` and `$subtitle`. While create the Open AI request, `$title` and `$subtitle` will be replaced by the actual value of `title` and `subtitle`.

# Why did you build Cloud Browser?

1. I liked the technical challenge. 
2. I've always wanted a better interface to view my AWS resources.
3. Maybe this will be *it* for some definition of _it_
