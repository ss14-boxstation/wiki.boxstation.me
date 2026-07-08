# SynthChem - Metabolism Rework
## The Current System, and its flaws
### Fundamental Flaws
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
### Incompatibility with New WizDen Metabolism
On top of all of that, WizDen's new metabolism changes the underlying idea for metabolism groups from categories that describe the intent of the chemical to the manner and location where they are processed.
While this doesn't prevent a "Software" metabolism stage to be created, it does complicate the filtering system and chems that are supposed to affect IPCs physically while still affecting Organics (such as acids and razorium).
## The incoming system, and how to adapt to it
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
```
## Code Overview
