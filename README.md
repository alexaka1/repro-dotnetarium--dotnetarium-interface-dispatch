# Dotnetarium disposal-analysis performance reproduction

Building this project with `Dotnetarium.Analyzers` 2.3.0 causes repeated analysis of incompatible disposal targets. The analyzer runs with its default configuration and call depths.

## Reproduce

Requires the .NET 10 SDK. Run the analyzer on a fresh compilation:

```sh
dotnet restore InterfaceDispatchRepro/InterfaceDispatchRepro.csproj
time dotnet build InterfaceDispatchRepro/InterfaceDispatchRepro.csproj --no-restore --no-incremental -p:UseSharedCompilation=false -p:ReportAnalyzer=true -v:minimal
```

The compiler consumes CPU while analyzing the disposal methods. The application does not need to run. No diagnostic is required to trigger the slowdown.

Each class has a query-bound string property and an `IJSObjectReference` supplied through its constructor. Its disposal method reads the property, then awaits JavaScript-reference disposal. The query-property read starts the XSS analysis. Each distinct disposal body supplies another candidate for interface dispatch.

## Control comparison

Remove `, IAsyncDisposable` from all 14 class declarations in `DisposalComponents.cs` and repeat the build. Keep the property reads, disposal calls, and package reference intact.

This removes the classes from the candidate set for `IAsyncDisposable.DisposeAsync`. The recognized input sources and analyzer configuration remain unchanged. Restore the original declarations after the comparison:

```sh
git restore InterfaceDispatchRepro/DisposalComponents.cs
```

## Measured results

Wall-clock measurements used SDK 10.0.401 on macOS arm64 with 18 logical processors. They used a fresh copy of this project and a fresh package cache containing the released 2.3.0 package. Both builds used default analyzer settings and the build options shown above. Restore time is excluded.

| Source variant | Result |
| --- | --- |
| Components implement `IAsyncDisposable` | Still compiling at 900 seconds; canceled. |
| Only `, IAsyncDisposable` removed from declarations | Build succeeded in 1.268 seconds, with zero warnings and errors. |

The canceled run establishes a lower bound; its eventual completion time is unknown. Timings depend on the machine. The control retains all query-input reads and JavaScript-reference disposal calls.

## Incorrect dispatch

`IJSObjectReference` inherits `IAsyncDisposable`. The expression `_module.DisposeAsync()` therefore refers to a method declared by `IAsyncDisposable`, while its receiver is an `IJSObjectReference`.

[GetInterfaceTargets in 2.3.0](https://github.com/dotnetarium/dotnetarium/blob/25616648a7b766af5f599d8a33cbc995421e7930/Roslyn/FlowAnalysis/FlowAnalysis/Analysis/TaintedDataAnalysis/TaintedDataAnalysis.TaintedDataOperationVisitor.cs#L608-L643) selects source implementations of the declaring interface. It does not require them to be compatible with the receiver. All 14 component disposal methods become candidates, although none of the classes implements `IJSObjectReference`.

Every selected method contains another JavaScript-reference disposal call. From the source code, this allows analysis to branch into the same candidate set repeatedly until the default call-depth limit. A managed-stack snapshot shows nested interprocedural analysis and interface-target discovery during the slow build.

Analysis should consider implementations compatible with the receiver's interface and complete promptly for this small project.
