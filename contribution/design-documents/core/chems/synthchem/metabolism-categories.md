---
title: SynthChem - Metabolism Rework
description: 
published: true
date: 2026-07-09T10:08:56.196Z
tags: 
editor: markdown
dateCreated: 2026-07-08T08:10:50.993Z
---

## *This Design Doc is unreviewed and has been merged as a test and example. Its contents are intended to be taken seriously but are subject to change*

| Designers                     | Coders | Implemented | GitHub Links    |
| ----------------------------- | ------ | ----------- | --------------- |
| CollectionOfBones128 (GitHub) | CollectionOfBones128 (GitHub)    | ℹ️ Open PR  | https://github.com/ss14-boxstation/ss14-boxstation/pull/195 |
# The Current System, and its flaws
## Fundamental Flaws
*Current Organ code, Pre New Metabolsim*
```yaml
# Organics
## Kidneys - Filter out SynthChems
  components:
  - type: Sprite
    layers:
      - state: kidney-l
      - state: kidney-r
  - type: Item
    size: Small
    heldPrefix: kidneys
  # The kidneys just remove anything that doesn't currently have any metabolisms, as a stopgap.
  - type: Metabolizer
    maxReagents: 5
    metabolizerTypes: [Human]
    removeEmpty: true
    groups: # Box Change
    - id: Software
      filter: true
    - id: Malware
      filter: true

# IPC
## Processor - Brain that also processes SynthChems
  components:
    - type: Metabolizer
      maxReagents: 2
      metabolizerTypes: [IPC, Synthetic]
      groups:
      - id: Software
      - id: Malware

## Micro Pump - Handles All organic chems
  components:
  - type: Metabolizer
    maxReagents: 2
    metabolizerTypes: [IPC, Synthetic]
    groups:
    - id: Industrial
    - id: Medicine
      filter: true
    - id: Poison
      filter: true
    - id: Narcotic
      filter: true
    removeEmpty: true
```
The system above is how it's currently done on BoxStation. Despite functioning well for all this time, it does have a few flaws.

First, if there is any new organ given to a species, that is extra maintainer load to ensure the filters are set correctly. And if any major overhauls to organs occurs, it requires a lot of work to fix.

Second, its highly inelegant. It requires duplicating a lot of code, such as if a chem is supposed to have identical or near-identical effects on Organic and Artificial lifeforms, and when defining all organ filters.

Third, it places a lot on the Players to remember. Despite Vox being allowed to 'use' SynthChem they don't metabolize Industrial, leading to any chem that is supposed to affect Vox & IPCs with a specific effect but not other Organics being required to use the "Drinks" metabolizer.

Finally, adding new metabolizer types at all risks creating a scenario where a chem gets stuck in an entities bloodstream. This would be exploitable to fully prevent
## Incompatibility with New WizDen Metabolism
On top of all of that, WizDen's new metabolism changes the underlying idea for metabolism groups from categories that describe the intent of the chemical to the manner and location where they are processed.
While this doesn't prevent a "Software" metabolism stage to be created, it does complicate the filtering system and chems that are supposed to affect IPCs physically while still affecting Organics (such as acids and razorium).
# The incoming system, and how to adapt to it
The new WizDen metabolism system provides a certain amount of advantages for plans going forwards. By moving the filtering of Organic & Artificial lifeform reagents to a more elegant and fundamental level and maintaining only the upcoming metabolism stages, it becomes easier to allow for the planned Synthethic & Cyberneticized traits.

Another fork with metabolizing IPCs, starcup, handles this by giving IPCs a Metabolizer whitelist and forcefully ejecting incompatible reagents. While this is effective and sidesteps some of the issues, it also does not seem like the ideal situation given the plans to allow more than just IPC & Vox to utilize SynthChem.

The proposal is simple: Utilize a system akin to tags and metabolizer types to organize all reagents and metabolizers under groups such as 'Organic' and 'Artificial'.

This system of 'Tags' or 'Categories' would also allow for seamless further expansion if desired, such as for 'Lithic' or 'Psionic' reagents & lifeforms.

While this system may seem redundant with the existing Metabolizer Types, the difference is simple: Utilizing Metabolizer Type Conditions would require editing every single reagent in the game. Avoiding this outcome has been a top priority since creating SynthChem.

The difference between Metabolizer Types and the proposed system also lands on a simpler element: The proposed system categorized the fundamental method by which an entity metabolizes their chems, whereas metabolizer types categorizes on a level below that: it captures the ways in which different organisms within the fundamental methods react to reagents.

*Proposed organ code, with New Metabolism*
```yaml
# IPC
## Micro Pump - Handles Bloodstream
  components:
  - type: Metabolizer
    maxReagents: 2
    metabolizerCategories: [ Artificial ] # While 'Synthetic' could be used here, disambiguation with Types may be ideal
    metabolizerTypes: [ Synthetic, Silicon ]
    stages: [ Bloodstream ]

# Vox
## OrganVoxMetabolizer - Parent of all Vox metabolizing organs
  components:
  - type: Metabolizer
    metabolizerCategories: [ Organic, Artificial ] # Rather than using a group for both Organics and Synthetics, parsing it as both at once is more ideal
    metabolizerTypes: [ Synthetic, Vox ]
```
*Proposed reagents with New Metabolism*
```yaml
# Organo-Synthetic
## Razorium (Abbreviated)
- type: reagent
  id: Razorium
  name: reagent-name-razorium
  group: Toxins
  slipData:
    requiredSlipSpeed: 3.5
  desc: reagent-desc-razorium
  physicalDesc: reagent-physical-desc-reflective
  flavor: sharp
  color: "#e3fffb"
  category: [ Organic, Synthetic ] # Will likely need some way to appear in the guidebook
  metabolisms:
    Bloodstream:
      metabolismRate : 3.00
      effects:
      - !type:HealthChange
        damage:
          types:
             Slash : 9
      - !type:PopupMessage
        type: Local
        visualType: LargeCaution
        messages: [ "generic-reagent-effect-slicing-insides"]
        probability: 0.33
      - !type:Emote
        conditions:
        - !type:MetabolizerTypeCondition
          type: [Silicon] # Using Metabolizer types, the effects of the chem can be made to suit IPCs without needing their own metabolizer type
          inverted: true
        emote: Scream
        probability: 0.3

# Synthetic only
## Weed.exe
- type: reagent
  id: WeedDotExe
  name: reagent-name-weed-dot-exe
  group: Narcotics
  desc: reagent-desc-weed-dot-exe
  physicalDesc: reagent-physical-desc-electric
  flavor: bitter
  flavorMinimum: 0.05
  color: "#30FB3E"
  category: [ Synthetic ] # Will likely need some way to appear in the guidebook
  metabolisms:
    Bloodstream:
      effects:
      - !type:ModifyStatusEffect
        effectProto: StatusEffectSeeingRainbow
        time: 16
        type: Add
```
# Code Overview
First and foremost, WizDen's new metabolism system has to first be merged. All code and pseudocode listed is based on the new metabolism system.

In essence, Reagents and Organs will be tagged with a Category (Organic, Artificial). All reagents will be metabolizable by all species but before checking if an effect can be applied, the code should check if *Any* of the Reagent's categories match *Any* of the Organ's categories.

A Species tagged with `[ Organic, Artifical ]` should be able to use all `[ Organic ]` chems *And* all `[ Artificial ]` chems.
A Reagent tagged with `[ Organic, Artificial ]` should be able to apply to Species that are only `[ Organic ]` *And* to Species that are only `[ Synthetic ]`.
## Prototypes
Content.Shared/Chemistry/Reagent/ReagentPrototype.cs
```c#
// Roughly L165
        /// <summary>
        /// Should this reagent work on the dead?
        /// </summary>
        [DataField]
        public bool WorksOnTheDead;
    
    // Box Change Start
	    /// <summary>
	    /// List of Metabolizer Categories this Reagent falls under
	    /// Defaults to Organic
	    /// </summary>
	    /// Pseudocode
	    [Data]
	    MetabolizerCategories = "Organic";
	// Box Change End
	    
        [DataField, AlwaysPushInheritance]
        public ReagentMetabolisms? Metabolisms;
```

Content.?/\_Box/Metabolism/MetabolizerCategoryPrototype.cs
```c#
// Pseudocode
// Will likely need to be made for Guidebook reasons? Doesn't seem to serve much purpose outside of that.
using Robust.Shared.Prototypes;

namespace Content.Shared._Box.Metabolism;

/// <summary>
/// Metabolizer Category identifier used to determine if a specific entity and a specific reagent are compatible.
/// </summary>
[Prototype]
public sealed partial class MetabolizerTypePrototype : IPrototype
{
    [IdDataField]
    public string ID { get; private set; } = default!;

    [DataField("name", required: true)]
    private LocId Name { get; set; }

    [ViewVariables(VVAccess.ReadOnly)]
    public string LocalizedName => Loc.GetString(Name);
```

## Components
Content.Shared/Metabolism/MetabolizerComponent.cs
```c#
// Roughly L73
    /// <summary>
    ///     Does this component use a solution on it's parent entity (the body) or itself
    /// </summary>
    /// <remarks>
    ///     Most things will use the parent entity (bloodstream).
    /// </remarks>
    [DataField]
    public bool SolutionOnBody = true;

// Box Change Start - SynthMetabolism
    /// <summary>
    ///    List of Metabolizer Categories that this organ can process (ex. Organic, Artifial)
    ///    Defaults to Organic
    /// </summary>
    /// <remarks>
    ///    Pseudocode
    /// </remarks>
    [Data]
    MetabolizerCategories = "Organic";
// Box Change End

    /// <summary>
    ///     List of metabolizer types that this organ is. ex. Human, Slime, Felinid, w/e.
    /// </summary>
    [DataField]
    [Access(typeof(MetabolizerSystem), Other = AccessPermissions.ReadExecute)] // FIXME Friends
    public HashSet<ProtoId<MetabolizerTypePrototype>>? MetabolizerTypes;

```
## Systems
Content.Shared/Metabolism/MetabolizerSystem.cs
```c#
//Roughly L190

            // if it's possible for them to be dead, and they are,
            // then we shouldn't process any effects, but should probably
            // still remove reagents
            if (isDead && !proto.WorksOnTheDead)
                continue;
                
            // Box Change Start
            // Pseudocode
            var compatibleMetabolism = (at least one reagent category matches at least one metabolizer category)
            // Box Change End

            var actualEntity = ent.Comp2?.Body ?? solutionOwner.Value;

            // do all effects, if conditions apply
            foreach (var effect in entry.Effects)
            {
                if (scale < effect.MinScale)
                    continue;

                if (rand.NextFloat() >= effect.Probability)
                    continue;

                // See if conditions apply
                if (effect.Conditions != null && !CanMetabolizeEffect(actualEntity, ent, solutionEntity.Value, effect.Conditions))
                    continue;
                    
                // Box Change Start - Pseudocode
                if (no compatibleMetabolism)
		            don't apply effect;
                // Box Change End

                ApplyEffect(effect);

            }
```