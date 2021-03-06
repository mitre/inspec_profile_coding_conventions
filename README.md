# InSpec Profile Coding Conventions


## Addressing Tests that can only be performed Manually at this time

Sometimes, tests described by the security guidance **cannot** really be automated well, and hence are can only be tested "manually", i.e. though human examination and/or interview.<br>
Such tests are to be setup with a skip statement to communicate to the reviewers that these tests are manual.

*Sample describe code for manual tests*
```
    describe 'This test can only be performed by manual examination or interview at this time.' do
      skip 'This test can only be performed by manual examination or interview at this time.'
    end
```
Tests configured this way are counted as `Not Reviewed` on Heimdall and inspec_tools and hence separated from actual failures. It allows the users to spot these guidance items as separate items to review manually.

## Use InSpec resources wherever applicable

When writing InSpec code to query different types of components, favor the use of Inspec resources wherever possible, instead of using a shell command with command resource. Tests written using InSpec resources tend be more reliable and support better documentation.

InSpec resources are modules within the InSpec core that are able to perform queries on a specific target platform, application, or a files.

https://www.inspec.io/docs/reference/resources/


## Use Input Values wherever applicable (i.e., avoid hard-coding values and parameters)

Favor the use input values to whenever applicable to pass run time values and compliance parameters over hard-coding values to the test.
This will allow for dynamic testing as well as the allow you to test against custom compliance requirements for your organization.

Input values can also be setup to pull values for ENV vars using ERB reference, this is useful to perform credentialed scans without having to code credentials into the profile.

*inspec.yml:*
```
  - name: password
    type: string
    required: true
    default: '<%=ENV['ARCHER_API_PASSWORD']%>'
```

More details:
https://www.inspec.io/docs/reference/inputs/

## Handling Sensitive Data with InSpec Resources
In some scenarios, you may be writing tests that involve resources with sensitive content, such as a file resource. 
In case of failures, you may desire to suppress output. Do this task by adding the :sensitive flag to the resource definition:

```
describe file('/tmp/mysecretfile'), :sensitive do
  its('content') { should match /secret_info/ }
end
```

## Ensure that Results Reporting is Descriptive

Tests should be designed in such a way that reporting of the status of the test be as descriptive of the intent of the test. 
This will avoid the need for checking the test DSL code or the test meta-data to make sense of the results.

If you are using target specific resources instead of a shell script, Inpsec already provides meaningfully descriptive status reports.
However if a command resource is used to perform query using a shell script, the `subject` feature can be used to allow better reporting.

The following example tests to validate the trusted users in the docker group on the target host demoing non-optimized and optimized reporting.

*Example of Vague Reporting*
```
describe  group('docker').members.split(',') do
  it { should be_in input('trusted_docker_users') }
end

Report generated:

  ["user1", "user2", "user3"]
     ×  is expected to be in "user1,user2"
     expected `["user1", "user2", "user3"]` to be in the list: `["user1,user2"]`
     Diff:
      ["user3"]
```
*Note that above above report provides no context on the test being performed.*

*Example of More Descriptive Reporting:*
```
describe 'Trusted Docker group members' do
  subject { group('docker').members.split(',') }
  it { should be_in input('trusted_docker_users') }
end

Report generated:

  Trusted Docker group members
     ×  is expected to be in "user1,user2"
     expected `["user1", "user2", "user3"]` to be in the list: `["user1,user2"]`
     Diff:
      ["user3"]

```

## Handling "Not Applicable" Tests

Sometimes, guidance will cite situations whereby a test is not applicable. For example, the following test to require the display of the graphical logon warning banner is deemed not applicable when the graphical interface is not installed:

These types of situations are coded with an `impact 0`. 

This is so that the test could still return the results of the test, yet the failure **does not** count against the overall compliance score.

Tests with `impact 0` are counted as `Non-Applicable` in `Heimdall` and `inspec_tools` compliance checker and stigchecklist converted 

Hence, if the check text of the guidance describes a condition for non-applicability, use an if condition to update impact based on the condition:

*DISA RHEL7 STIG Inspec Profile
V-71893:*

```
Checktext:
...
Note: If the system does not have GNOME installed, this requirement is Not
Applicable.
...
```

*Conditional impact level:*

https://github.com/simp/inspec-profile-disa_stig-el7/blob/52fd7c3cd647e8b7d50478dbb6ca7bdcb7fc35e4/controls/V-71893.rb#L17-L21
```
  if package('gnome-desktop3').installed?
    impact 0.5
  else
    impact 0.0
  end
```

## Handling Virtualized vs not situations:

If a tests needs to executed differently or change the impact based on whether the target is virtualized or not, the following resources should be used:

- [virtualization](https://www.inspec.io/docs/reference/resources/virtualization/) : Return the virtualization details of the target
- [os](https://www.inspec.io/docs/reference/resources/os/) : Return the OS details of the target


*Container specific execution*
```
if virtualization.system.eql?('docker')
  # Container specific tests
else
  # regular host tests
end
```

## "Contain the Code!": 

Contain Edge Cases/Conditions where an InSpec ("describe block") test might not run:

For example, if the InSpec DSL describe blocks are contained within a condition block or a loop, there is a chance that the execution does not reach the describe block and hence **no results will be returned** for the test.

If tests are not contained, the "results", or lack thereof, will be marked as `Profile Errors` on Heimdall.

Ensure that tests cover such cases as in the example shown below.

*Ref: https://github.com/mitre/cis-aws-foundations-baseline/blob/9409f7a3a87337961db371a101929f569e981972/controls/cis-aws-foundations-1.23.rb#L86-L98*
```
  aws_iam_access_keys.entries.each do |key|
    describe key.username do
      context key do
        its('last_used_days_ago') { should_not be_nil }
      end
    end
  end

  if aws_iam_access_keys.entries.empty?
    describe 'Test skipped because no iam access keys were found' do
      skip 'This test is skipped since the aws_iam_access_keys resource returned an empty access key list'
    end
  end
```

The `inspec check` command can be useful in tracking guidance with no tests or unexposed describe blocks.

## Run Optimization

In cases compliance checks has to be performed over a very large array set, such as file properties of all the files in a directory, there is chance the test run takes an inordinate amount of to complete.
In such cases it is recommended to see if a shell command on the target can be leveraged to speed up the test.

*https://github.com/simp/inspec-profile-disa_stig-el7/blob/52fd7c3cd647e8b7d50478dbb6ca7bdcb7fc35e4/controls/V-72009.rb#L38-L43*
```
Check text: "Verify all files and directories on the system have a valid group.
If any files on the system do not have an assigned group, this is a finding."

  command('grep -v "nodev" /proc/filesystems | awk \'NF{ print $NF }\'').
    stdout.strip.split("\n").each do |fs|
      describe command("find / -xautofs -fstype #{fs} -nogroup") do
        its('stdout.strip') { should be_empty }
      end
    end

```

## Check for "Profile Error"

If an Inspec test execution does not reach a describe block or if a runtime exception is encountered, the test will be counted as "Profile Errors" on `Heimdall` and `inspec_tools`. 

Review you InSpec results JSON in Heimdall to verify there are no "Profile Errors" present.

The only acceptable "Profile Errors" are the ones resulted from incorrect privileges during the InSpec run.

## Check for risky commands (e.g. rm, del, purge, etc.)

Validate that risky commands are not being used on the profile that could perform damage or change to the target.

## Put in place a profile review process

Ensure a profile review process is in place to ensure the best practices are adhered to during the development for the profiles.
A template for profile review used by MITRE is available here.

https://github.com/mitre/microsoft-iis-8.5-server-stig-baseline/blob/master/Review.md


# License and Author

### Authors

- Author:: Rony Xavier [rx294](https://github.com/rx294)
- Author:: Eugene J Aronne [ejaronne](https://github.com/ejaronne)

### NOTICE

© 2018 The MITRE Corporation.

Approved for Public Release; Distribution Unlimited. Case Number 18-3678.

### NOTICE

MITRE hereby grants express written permission to use, reproduce, distribute, modify, and otherwise leverage this software to the extent permitted by the licensed terms provided in the LICENSE.md file included with this project.

### NOTICE

This software was produced for the U. S. Government under Contract Number HHSM-500-2012-00008I, and is subject to Federal Acquisition Regulation Clause 52.227-14, Rights in Data-General.

No other use other than that granted to the U. S. Government, or to those acting on behalf of the U. S. Government under that Clause is authorized without the express written permission of The MITRE Corporation.

For further information, please contact The MITRE Corporation, Contracts Management Office, 7515 Colshire Drive, McLean, VA 22102-7539, (703) 983-6000.

