# Monolith Plugin UE 5.5.4 Adaptation Log

## Plugin Info
- **Original Requirement**: UE 5.7+
- **Adapted For**: UE 5.5.4
- **Date**: 2026-04-21

## UE Version Macros
```cpp
// Recommended pattern for version guards
#if ENGINE_MAJOR_VERSION >= 5 && ENGINE_MINOR_VERSION >= 7
    // UE 5.7+ code (original)
#else
    // UE 5.5 fallback
#endif
```

---

## UE 5.5 Feature Gaps (From 5.7 Downgrade)

The following features from UE 5.7 are **NOT available** in UE 5.5:

### Completely Disabled Modules
| Module | Reason |
|--------|--------|
| StateTree Actions | StateTreeExtension.h missing, API completely changed |
| MetaSound Write Actions | MetasoundFrontendNodeClassRegistry.h missing, Builder API changed |
| PoseSearch Write Actions | Channel headers (Position/Velocity/etc.) moved to Private, not exported |

### API Not Exported (Link Errors)
| API | Module | Impact |
|-----|--------|--------|
| `UK2Node_LatentAbilityCall` | GAS | Ability task node creation unavailable |
| `USoundNodeOscillator` | Audio | Sound Cue oscillator node unavailable |
| `USoundNodeDoppler` | Audio | Sound Cue doppler node unavailable |
| `FNiagaraStackGraphUtilities::GetStackFunctionInputs` | Niagara | Uses fallback pin iteration |
| `FNiagaraStackGraphUtilities::GetStackFunctionStaticSwitchPins` | Niagara | Uses fallback pin iteration |
| `FNiagaraStackGraphUtilities::SetLinkedParameterValueForFunctionInput` | Niagara | Module input linking degraded |
| `UMaterialExpressionCustom::RebuildOutputs` | Material | Outputs may need manual refresh |

### Protected/Hidden Methods
| API | Module | Impact |
|-----|--------|--------|
| `FMassEntityManager::GetMatchingArchetypes` | AI | Mass archetype query unavailable |
| `UIKRigDefinition::GetSolverStructs` | Animation | Solver type info degraded |
| `UIKRigController::GetStartBone/SetStartBone` | Animation | IK solver root bone setting unavailable |
| `UIKRigController::AddSolver(FString)` | Animation | Uses TSubclassOf fallback |

### Changed Signatures
| API | 5.5 Signature | 5.7 Signature |
|-----|---------------|---------------|
| `UGameplayTagsManager::RenameTagInINI` | 2 params | 3 params (bBroadcastChanges) |
| `USmartObjectDefinition::Validate` | `TArray<FText>*` | `TArray<TPair<EMessageSeverity,FText>>*` |
| `UMaterialInterface::GetUsedTextures` | 5 params | 2 params |
| `UStaticMesh::GetNaniteSettings` | Not exported | Returns struct |

### Missing Properties (5.7+ only)
| Property | Module |
|----------|--------|
| `UMaterialFunctionInstance::ParameterCollectionParameterValues` | Material |
| `UMaterialExpressionFunctionInput::BlendInputRelevance` | Material |
| `UWidgetBlueprint::WidgetVariableNameToGuidMap` | UI |
| `UWidgetBlueprint::OnVariableAdded/OnVariableRemoved` | UI |
| `FWidgetBlueprintEditorUtils::EDeleteWidgetWarningType::DeleteSilently` | UI |

---

## Modifications Summary (33 fixes)

### Round 1 - Basic Compatibility

**Mod #1: MaterialDomain.h Include**
- Files: MonolithMesh (3 files), MonolithMaterialActions.cpp
- Issue: 5.5 Material.h only forward-declares EMaterialDomain

**Mod #2: GetMaterialResource API**
- File: MonolithMaterialActions.cpp
- 5.5: Takes ERHIFeatureLevel, 5.7: Takes EShaderPlatform

**Mod #3: UK2Node_SwitchEnum::SetEnum**
- File: MonolithBlueprintNodeActions.cpp
- Fix: Use INodeDependingOnEnumInterface::ReloadEnum (not exported in 5.5)

**Mod #4: FJsonSerializer::Serialize**
- Files: MonolithAIIndexer.cpp, DataAssetIndexer.cpp, GASIndexer.cpp
- 5.5 requires TSharedRef, not TSharedPtr

### Round 2 - Conditional Headers

**Mod #5-10: Conditional Includes**
- StateTreeExtension.h → Entire file disabled
- SmartObjectRequestTypes.h → Conditional include
- IKRetargetChainMapping.h → Conditional include
- CancelAbilityTagsGameplayEffectComponent.h → Conditional include
- MetasoundFrontend headers → Alternate include
- Sequencer module dependency added

### Round 3 - Complete Fix (Build Success)

**Mod #11-31: Full API Adaptation**
See detailed list in [Round 3 Modifications](#round-3-modifications) section.

### Round 4 - Improvements

**Mod #32: ApplyPerlinNoiseToMesh Version Isolation**
- File: MonolithMesh/Private/MonolithMeshProceduralActions.cpp
- Issue: `ApplyPerlinNoiseToMesh2` is 5.7+ enhanced version with more noise options
- Fix: Use version guard to preserve 5.7 noise quality
```cpp
#if ENGINE_MAJOR_VERSION >= 5 && ENGINE_MINOR_VERSION >= 7
    UGeometryScriptLibrary_MeshDeformFunctions::ApplyPerlinNoiseToMesh2(...);
#else
    UGeometryScriptLibrary_MeshDeformFunctions::ApplyPerlinNoiseToMesh(...);
#endif
```

**Mod #33: Widget Deletion Variable Cleanup**
- File: MonolithUI/Private/MonolithUIActions.cpp
- Issue: 5.5 `RemoveWidget` doesn't clean up variable nodes, may leave orphaned variables
- Fix: Manual cleanup for 5.5
```cpp
#if ENGINE_MAJOR_VERSION >= 5 && ENGINE_MINOR_VERSION >= 7
    FWidgetBlueprintEditorUtils::DeleteWidgets(WBP, WidgetsToDelete, EDeleteWidgetWarningType::DeleteSilently);
#else
    if (Widget->bIsVariable)
    {
        FBlueprintEditorUtils::RemoveVariableNodes(WBP, Widget->GetFName());
    }
    WBP->WidgetTree->RemoveWidget(Widget);
#endif
```

---

## Round 3 Modifications (Detailed)

| # | Module | Fix |
|---|--------|-----|
| 11 | GAS | Syntax fix - duplicate return statement |
| 12 | Audio | MetaSound RegisterActions structure fix |
| 13 | Material | BlendInputRelevance version guard |
| 14 | Material | ParameterCollectionParameterValues version guard |
| 15 | Material | RenameMaterialFunctionParameterGroup version guard |
| 16 | Material | RebuildOutputs version guard (not exported) |
| 17 | Mesh | ApplyPerlinNoiseToMesh2 version isolation |
| 18 | Mesh | GetNaniteSettings fallback to direct property |
| 19 | Mesh | GetUsedTextures signature fix (5 params) |
| 20 | AI | GetMatchingArchetypes disabled (protected) |
| 21 | AI | SmartObject Validate signature adaptation |
| 22 | UI | WidgetBlueprint properties version guard |
| 23 | Animation | IKRig API version guards |
| 24 | Animation | FVector2f → FVector2D |
| 25 | Animation | ControlRig GetRigVMClient direct access |
| 26 | Animation | PoseSearch entire file disabled |
| 27 | GAS | UK2Node_LatentAbilityCall version guards |
| 28 | GAS | RenameTagInINI signature fix |
| 29 | Niagara | SetLinkedParameterValueForFunctionInput version guard |
| 30 | Niagara | GetStackFunctionInputs/StaticSwitchPins compatibility wrappers |
| 31 | Audio | SoundNodeOscillator/Doppler version guard |

---

## Build Status

**Result**: ✅ ALL 11 MODULES COMPILE SUCCESSFULLY

| Module | Status |
|--------|--------|
| MonolithCore | ✅ |
| MonolithBlueprint | ✅ |
| MonolithGAS | ✅ |
| MonolithAI | ✅ |
| MonolithMaterial | ✅ |
| MonolithMesh | ✅ |
| MonolithAnimation | ✅ |
| MonolithUI | ✅ |
| MonolithAudio | ✅ |
| MonolithNiagara | ✅ |
| MonolithIndex | ✅ |

---

## Key Patterns Used

### Pattern 1: Conditional Include
```cpp
#if ENGINE_MAJOR_VERSION >= 5 && ENGINE_MINOR_VERSION >= 7
#include "HeaderOnlyIn57.h"
#endif
```

### Pattern 2: Conditional Code Block
```cpp
#if ENGINE_MAJOR_VERSION >= 5 && ENGINE_MINOR_VERSION >= 7
    // 5.7+ API
#else
    // 5.5 fallback or error
#endif
```

### Pattern 3: Compatibility Wrapper
```cpp
namespace MonolithHelpers
{
    void APICompat(...)
    {
    #if ENGINE_MAJOR_VERSION >= 5 && ENGINE_MINOR_VERSION >= 7
        EngineAPI::Function(...);
    #else
        // Fallback implementation
    #endif
    }
}
```

### Pattern 4: Entire File Disable
```cpp
#if ENGINE_MAJOR_VERSION >= 5 && ENGINE_MINOR_VERSION >= 7
// Full implementation
#else
void RegisterActions(FMonolithToolRegistry& Registry) {}
FMonolithActionResult HandleAction(...)
{
    return Error("Requires UE 5.7+");
}
#endif
```

---

## Modified Files List

```
Source/
├── MonolithMesh/Private/
│   ├── MonolithMeshBlockoutActions.cpp      (Mod #1)
│   ├── MonolithMeshDecalActions.cpp         (Mod #1)
│   ├── MonolithMeshDebugViewActions.cpp     (Mod #1)
│   ├── MonolithMeshInspectionActions.cpp    (Mod #18)
│   ├── MonolithMeshTechArtActions.cpp       (Mod #19)
│   └── MonolithMeshProceduralActions.cpp    (Mod #17, #32)
│
├── MonolithMaterial/Private/
│   └── MonolithMaterialActions.cpp          (Mod #1, #2, #13-16)
│
├── MonolithBlueprint/Private/
│   └── MonolithBlueprintNodeActions.cpp     (Mod #3)
│
├── MonolithAI/Private/
│   ├── MonolithAIIndexer.cpp                (Mod #4)
│   ├── MonolithAIStateTreeActions.cpp       (Mod #5)
│   ├── MonolithAIRuntimeActions.cpp         (Mod #6)
│   ├── MonolithAIAdvancedActions.cpp        (Mod #20)
│   └── MonolithAISmartObjectActions.cpp     (Mod #21)
│
├── MonolithGAS/Private/
│   ├── MonolithGASEffectActions.cpp         (Mod #8, #11)
│   ├── MonolithGASTargetActions.cpp         (Mod #27)
│   ├── MonolithGASAbilityActions.cpp        (Mod #27)
│   └── MonolithGASTagActions.cpp            (Mod #28)
│
├── MonolithAnimation/Private/
│   ├── MonolithAnimationActions.cpp         (Mod #7, #23, #24)
│   ├── MonolithAbpWriteActions.cpp          (Mod #24)
│   ├── MonolithControlRigWriteActions.cpp   (Mod #25)
│   └── MonolithPoseSearchActions.cpp        (Mod #26)
│
├── MonolithUI/Private/
│   ├── MonolithUIInternal.h                 (Mod #22)
│   ├── MonolithUIActions.cpp                (Mod #22, #33)
│   └── MonolithUIAnimationActions.cpp       (Mod #22)
│
├── MonolithAudio/Private/
│   ├── MonolithAudioMetaSoundActions.cpp    (Mod #9, #12)
│   └── MonolithAudioSoundCueActions.cpp     (Mod #31)
│
├── MonolithNiagara/Private/
│   ├── MonolithNiagaraActions.cpp           (Mod #29, #30)
│   └── MonolithNiagara.Build.cs             (Mod #10)
│
└── MonolithIndex/Private/Indexers/
    ├── DataAssetIndexer.cpp                 (Mod #4)
    └── GASIndexer.cpp                       (Mod #4)
```

---

## Notes for Future UE Version Changes

1. **Always check export macros** - NIAGARAEDITOR_API, UNREALED_API, etc. may change between versions
2. **Private headers** - Plugins often move implementation headers to Private, breaking external access
3. **Signature expansion** - New versions often add optional parameters to existing functions
4. **Protected methods** - Functions may become protected, requiring reflection or interface workarounds
5. **Test with actual project** - Build verification is essential, LSP errors are misleading in cross-version scenarios