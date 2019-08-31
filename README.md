# Core network stacks

## Automatic detection of third AZ

Each VPC stack contains logic to detect if a third AZ is present in the region, and create a subnet and associated resources in that AZ accordingly.

To achieve this, each VPC stack is passed a comma-delimited list of availability zones in the *AzList* stack parameter:

    Parameters:
      AzList:
        Description: Comma delimited list of AZs in this region
        Type: String

The VPC stacks are created as substacks from the top-level network stack, and passed the current region's comma-delimited list of AZs:

    Vpc:
      Type: AWS::CloudFormation::Stack
      Properties:
        TemplateURL: "..."
        Parameters:
          AzList: { "Fn::Join": [ ",", "Fn::GetAZs": "" ] }

The *AzList* parameter value is used in the Conditions section of the VPC substack stack with the following logic:

    Conditions:
      ThreeAzs: { "Fn::Not": [ "Fn::Equals": [ "Fn::Select": [ 2, "Fn::Split": [ ",",  "Fn::Sub": "${AzList},," ] ], "" ] ] }

This condition logic:

* Appends ",," to the end of the AzList value
* Splits the appended AZ list by comma (value becomes: )
* Selects the third element from the list (selected item is: "")
* If the third element is blank, then there is no third AZ.

The condition is then used to determine if resources should be created in the third AZ.

Note: We need to pass in *AzList* to the stack, because Fn::GetAZs is not allowed in a condition statement

### Example (two-AZ region)

* AzList parameter is `ap-southeast-1a,ap-southeast-1b`
* ",," is appended to the end of AzList => `ap-southeast-1a,ap-southeast-1b,,`
* List is split by comma into an array: `[ "ap-southeast-1a", "ap-southeast-1b", "", ""]`
* The third element is selected => `""`
* The selected element is blank => ThreeAzs condition is false

### Example (three-AZ region)

* AzList parameter is `ap-southeast-2a,ap-southeast-2b,ap-southeast-2c`
* ",," is appended to the end of AzList => `ap-southeast-2a,ap-southeast-2b,ap-southeast-2c,,`
* List is split by comma into an array: `[ "ap-southeast-2a", "ap-southeast-2b", "ap-southeast-2c", "", ""]`
* The third element is selected => `ap-southeast-2c`
* The selected element is **not** blank => ThreeAzs condition is true

### Todo
* Add some simple monitoring for direct connect


### To Do a deployment from a laptop in an emergency

    export AWS_PROFILE=abc-automation
    export AWS_DEFAULT_REGION=ap-southeast-1
    cd ~/abc/git/abc-core-automation/
    bash bin/deployspec-package-and-upload.sh abc core network test 1
    cd lambdas/deployspec_compiler/
    python3.6 test.py abc core network test 1
    cd ../execute
    python3.6 simulate.py deploy abc core network test 1

