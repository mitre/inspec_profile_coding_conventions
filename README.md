# InSpec Profile Coding Conventions


## Addressing Manual tests

Some of the checks described by the security guidance **cannot** be automated and hence are manaul.<br>
Such checks are to be setup with a skip statement to communicate to the reviwers that these tests are manual.

*Skip statement for manual controls*
```
    describe 'This control is skipped since this is a manual test' do
      skip 'This control is skipped since this is a manual test'
    end
```
Controls with skip statements are bucketed as `Not Reviewed` on Heimdall and inspec_tools and hence separated from actual failures.

## Use Input Values wherever applicable

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

## Sensitive resources
In some scenarios, you may be writing checks that involve resources with sensitive content, such as a file resource. 
In case of failures, you may desire to suppress output. Do this task by adding the :sensitive flag to the resource definition:

```
describe file('/tmp/mysecretfile'), :sensitive do
  its('content') { should match /secret_info/ }
end
```

## Reporting Optimization

Tests should be designed in such a way that reporting of the status of the check be as descriptive of the intent of the test. 
This will avoid the need for checking the test DSL code or the control meta-data to make sense of the results.

If you are using target specific resources instead of a shell script, Inpsec already provides meaningfully descriptive status reports.
However if a command resource is used to perform query using a shell script, the `subject` feature can be used to allow better reporting.

The following example checks to validate the trusted users in the docker group on the target host demoing non-optimized and optimized reporting.

*Non-Optimized Reporting*
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

*Report Optimized test:*
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

## Non Applicable Controls

If a specific control is deemed to be `Non-applicable` on the target, are coded with an `impact 0`. 

This is so that the control could still return the results of the test, yet the failure **does not** count against the overall compliance score.

Controls with `impact 0` are bucketed as `Non-Applicable` in `Heimdall` and `inspec_tools` compliance checker and stigchecklist converted 


## Conditional Non Applicability

If the check text of the Control guidance describes a condition for non-applicability, use an if condition to update impact based on the condition.

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

## OS/Platform based execution

If a controls needs to executed diffrently or impact updated based on Virtualization or OS the following resource should be used.

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

## Cover edge conditions where test might not run where describe is within a loop/condition

If the InSpec DSL describe blocks are contained within a condition block or a loop, there is a chance that the execution does not reach the describe block and hence **no results will be returned** for the control.
These controls will be bucketed as `Profile Errors` on Heimdall.

Ensure that controls cover such cases as in the example shown below.

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
    describe 'Control skipped because no iam access keys were found' do
      skip 'This control is skipped since the aws_iam_access_keys resource returned an empty access key list'
    end
  end
```

The `inspec check` command can be useful in tracking controls with no tests or unexposed describe blocks.

## Run Optimization

In cases compliance checks has to be performed over a very large array set, such as file properties of all the files in a directory, there is chance the control run takes an inordinate amount of to complete.
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

## Put in place a profile review process

Ensure a profile review process is in place to ensure the best practices are adhered to during the development for the profiles.
A template for profile review used by MITRE is available here.

https://github.com/mitre/microsoft-iis-8.5-server-stig-baseline/blob/master/Review.md
