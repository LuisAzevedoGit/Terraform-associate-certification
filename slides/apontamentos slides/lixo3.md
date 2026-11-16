# Aula 03 - provider versioning and the lock file

constraint says what is acceptable. the lock file says what was chosen.


## Provider use semantic versioning

Provider versions move fast. a widely cloud provider may realease several times a month.
THis is why constrains matter. without one, terraform accepts any version.
version argument is optional, but strongly recomended.
Terraform selects the newest version that satisfies every constraint

on the exam:
Constraints from the root module and every child module are combined. all of them must be satisfied by one selected version


= 1.4.0 or just 1.4.0 | exactly this verison
!= 1.4.0 | any except this
>= 1.2.0 | this or newer
<= 1.2.0 | this or older
> 1.2.0 and < 2.0.0 | range
~> 1.4.0 | pessimistic. allows the rightmost part to increase, nothing above it.

exam: comma-separated constraints are combined with and, never or. every one must be satisfied.


## pessimistic operator

example: ~> 1.4.0  accepts 1.4.1, 1.4.2... and 1.4.9. rejects 1.5.0
     ~> 1.4 accepts 1.5, 1.6 ... and 1.9. rejects 2.0

## example

chosen from 1.4.0, 1.4.7, 1.5.2 , 2.0.1

~> 1.4.0 --> 1.4.7
~> 1.4 --> 1.5.2
>= 1.4.0 --> 2.0.1 (terraform always chose the newest that satisfies every constraint)
No version argument --> 2.0.1


## required_version or required_providers
required version:
    constraints terraform itself
        sits in terraform block
        one setting only
        checked before anything else runs

required_providers version
    Constrains a plugin:
        sits inside required_providers
        one entry per provider
        resolved during terraform init


Both live in the terraform block and  both take some operators. THey constrain completely different software.

## where to be strict and where not to be
