---
layout: page
title: DevOps - Pipeline Tweaks
parent: DevOps
---

# Pipeline Tweaks

Depending on the size and number of projects, build output files and place to run the build you can get a acceptable fast pipeline or be slow and have to wait on each build.
There are some methods to fasten up your pipe and optimize the single steps.


## MS DevOps CI (continuous integration)

The build process of the pipe could be run in the cloud in a container, or you can host the build process on a locally hosted system. I've made a comparison of both methods for a big solution with multiple projects at a company at work. We had to build multiple gRPC servers with multiple DLLs in each build output folder and prepare the artifacts to deploy all these servers each. We had 4 GB and about 5000 files as output. Steps like copying, publishing or zipping can get really slow on a size like this. The task to design the build to output each server deployment ready with all its DLLs slowed the pipeline from 2-5 min to about 22 min runtime. This was not acceptable, so we had to reduce the runtime and researched the following steps:


### Cloud Container vs On Premise Build Server

At first we have built the separated servers on our on Premise build server. The server had multiple virtual CPUs and 16 GB RAM. Building and restoring NuGet packages was fast. But downloading the source code to the local server, uploading the artifacts to the cloud again and then downloading them again to publish them on the on Premise target server seemed to be unneeded copy steps with limited internet bandwidth. 

So we decided to build in the cloud and use the suspected internal Azure bandwidth for the publishing to only have the limited bandwidth at releases. But we were disappointed to see that the limited container was very slow on restoring NuGet packages, on building, and even on publishing the artifacts back to the DevOps cloud storage. We had no speed gains like this so we needed to change the basic design of the pipeline.

Our next step was to research for more parallelism of the publishing step. We had the assumption that the `PublishBuildArtifacts@1` task was really slow. After some research we found the `PublishPipelineArtifact@1` task, which turned out to be really fast!

Another optimization was to reduce the needed task themselves, so we decided to skip the `CopyFiles@2` task for preparing the build outputs in the staging directory and publish directly out of our defined deployment folder. This folder path is stored in each project, separates the servers and makes the staging folder kind of obsolete.

The comparison of cloud container and on Premise server showed that the on Premise server was still faster after these optimization steps because it was faster at building and restoring. The difference was about 2-3 minutes.

All in all we have reduced the pipeline runtime from the devastating 22 minutes to blazing 3-4 minutes again! 

Here's the finished pipeline as YAML:

```yaml
trigger:
  branches:
    include:
      - '*'
    exclude:
      - 'main'
      - 'Beta'
      - 'Alpha'
      - 'Alpha*'

## DevOps Azure Container:
# pool:
#   vmImage: 'windows-latest'

## OnPremise Build machine:
pool:
  name: OnPremise Pipelines
  demands: 
    - Agent.Name -equals YOUR_BUILD_SERVER

variables:
  - name: SolutionName
    value: YOUR_SOLUTION_NAME
  - name: solution
    value: '**/*.sln'
  - name: buildPlatform
    value: 'Any CPU'
  - name: buildConfiguration
    value: 'Release'

steps:
- checkout: self
  submodules: true
  persistCredentials: true
  fetchDepth: 0

- task: NuGetToolInstaller@1
  displayName: 'Install NuGet Tools'
  inputs:
    checkLatest: true

- task: NuGetCommand@2
  displayName: 'Restore NuGet Packages'
  inputs:
    restoreSolution: '$(solution)'

- task: VSBuild@1
  displayName: 'Build Solution'
  inputs:
    solution: '$(solution)'
    platform: '$(buildPlatform)'
    configuration: '$(buildConfiguration)'

- task: CopyFiles@2
  displayName: 'Copy build artifacts to staging folder'
  inputs:
    SourceFolder: '$(System.DefaultWorkingDirectory)'
    Contents: '$(SolutionName)*/bin/$(buildConfiguration)/**'
    TargetFolder: '$(Build.ArtifactStagingDirectory)'

- task: PublishPipelineArtifact@1
  displayName: 'Publish Pipeline Artifacts'
  inputs:
    targetPath: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'
    publishLocation: 'pipeline'
```

## Conclusion

If you have a on Premise build server with some power and a good ISP it might be faster to use this compared to the free DevOps build container. NuGet Restore, publishing the artifacts and building could be way faster like this. 

Reducing steps of pipe to a minimum can bring some acceleration, too, if you have the possibility to do so. 
