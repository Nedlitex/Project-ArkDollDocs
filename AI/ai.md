# AI Design

## AI States

AI will always be in one of the two possible states: `Free` or `Busy`.

- `Busy` state means AI is currently performing some action that cannot be interrupted, for example
attacking an enemy or on the way to attack. AI will have an objective in this state and that objective
will not be changed unless it becomes invalid. AI will not re-evaluate the current state during `Busy`
state. AI should maximize its time in `Busy` state.
- `Free` state means AI is currently willing to re-evaluate the state and determine the next best action. Note that AI can still be performing some action (like patrolling) while in `Free` state. AI will periodically re-evaluate in `Free` state and try to determine the best action. AI should minimize its time in `Free` state.

---

## AI Actions

### Action Categories

The actions that AI can take has three categories: `Offensive`,  `Defensive` and `Reactive`.

- `Offensive` actions includes actions like attacking or using ability against the target.
- `Defensive` actions includes actions like keeping a distance to the the target (keep vigilance), or fleeing from the threat.
- `Reactive` actions includes actions like parry or dodge that often responses to the target's action.

Note that AI by default has an `Idle` action that is considered as a `Defensive` action.

When determining which action to take, AI will always choose the `Reactive` action if the conditions met. Otherwise, AI will first determine the category of action to take. AI has an algorithm that computes its `Aggressive Tendency` (`AT`), and the AI will perform an `Offensive` action if `AT` >= `AT_Threshold`. Otherwise, AI will pick a `Defensive` action. Note that if there is no `Offensive` action programmed for the AI, then it will always default to use `Defensive` actions.

### Aggressive Tendency

`AT` is computed with the following formula:

```
AT = AT_Base + Sum(AT_Offset) + T_Action * AT_Acc_Rate
```

Where:

- `AT_Base` is the base `AT` value that can be different based on the current HP percentage. In other words:

        AT_Base = HP_Percentage >= AT_HP_Threshold ? AT_Base_High : AT_Base_Low

    This design allows AI to be more aggressive when in high HP and more defensive when in low HP.

- `AT_Offset` is a value carried by each action. AI can specify a negative number for each `Offensive` action in order to prevent it from attacking forever.

- `AT_Acc_Rate` is a value representing the change to `AT` per second. `T_Action` represents the time spent in the current action. This can be used as a soft timer to force AI to do some `Offensive` action once a while.

- AI can also set a flag `at_reset` to reset the `AT` back to `AT_Base`.

### Action Priority

AI uses the following algorithm to determine the priority of the actions:

1. `Reactive` actions have the highest priority.

2. `Offensive` actions have higher priority than `Defensive` actions.

3. `Idle` action has the lowest priority among all actions.

4. For actions with the same category, `priority` attribute of the action is used to determine the priority.

### Action Charge System

AI has a `charge` value that can be affected by each action. An action can also specify a minimum required charge. As an example, AI can define several normal `Attack` actions that increment the `charge`, and an `Ultimate` `Attack` action that requires (and consumes) a certain `charge` level. Another example is to give a set of `Attack` actions an increasing level of `charge` requirements, with the finishing `Attack` consumes all the charges. This way AI can perform a repeated combo of actions.

### Event System

AI uses a event system to get notification from the environment that serves as the part of the conditions to each actions. Note that if AI has `Reactive` actions, events will be checked during each AI decision loop (every frame), whereas if the AI does not have `Reactive` action, events are only checked when AI is about to choose the next action.

Table below describes all the possible events:

`AI_EVENT`

| Name | Description |
| - | - |
| in_enemy_atk_range | This event is set if the AI is currently in the attack range of an enemy. Note that this value is an estimation of the range of the weapon that the enemy is currently using. |
| in_enemy_active_atk_range | This event is set if the AI is currently in the attack range of an attacking enemy. |
| in_target_atk_range | This event is set if the AI is currently in the attack range of its current target. Note that this value is an estimation of the range of the weapon that the target is currently using. This value implies `in_enemy_atk_range`. |
| in_target_active_atk_range | This event is set if the AI is currently in the attack range of an attacking enemy. This value implies `in_enemy_active_atk_range`. |
| enemy_atk_melee | This event is only set if `in_enemy_active_atk_range` is set, and the enemy is doing a melee attack. |
| enemy_atk_ranged | This event is only set if `in_enemy_active_atk_range` is set, and the enemy is doing a ranged attack. |
| enemy_in_atk_range | This event is set if there exists an enemy in AI's `atk_range_hint`. |
| target_in_atk_range | This event is set if the target is in AI's `atk_range_hint`. This value implies `enemy_in_atk_range`. |
| on_kill | This event is set if the AI killed an enemy in the previous frame. Note that this event generally is only useful for `Reactive` actions. |
| on_assist | This event is set if the AI assisted killing an enemy in the previous frame. Note that this event generally is only useful for `Reactive` actions. |
| on_hit | This event is set if the AI receives damage in the previous frame. Note that this event generally is only useful for `Reactive` actions. |
| on_dealt_damage | This event is set if the AI deals damage in the previous frame. Note that this event generally is only useful for `Reactive` actions. |
| on_recovered | This event is set if the AI recovered from an uncontrollable state (like stun) in the previous frame. Note that this event generally is only useful for `Reactive` actions. |

### Ticketing System

It is often the case that we want multiple AIs to surround their common target in order to create a more challenging situation for the player. A ticketing system is designed to achieve this. A `ticket` is a position around a target, and `tickets` are evenly spread around the target. AI action that uses the ticketing system will claim a `ticket` and use that as its reference position when approaching the target. If the action does not use `ticket`, AI will just find the shortest path to the target.

The ticketing system is also used to implement a flanking system. AI flanks the target by keep moving to the next ticket position.

It is generally recommended to use the ticketing system for mobs as it ensures a swamp of enemies can spread nicely around the player and reduce collisions during their path-findings. For a elite/boss enemy, it can choose to not use the ticketing system for a more natural interaction.

### Action Configuration

Table below defines the configuration available to AI animation.

`AI_ANIM`

| Key | Type | Range | Description | Default |
| - | - | - | - | - |
act_id | string | - | The action id of the animation. Action ids are defined under `/Data/Actions`. | "" |
combo_frame | int | [0, inf) | The frame number that AI should perform the next animation. 0 means at the end of the animation. | 0 |
combo_gap | float | [0, inf) | The time in seconds the AI should wait before transition to next action. | 0 |


Table below defines the configurations available to the AI action.

`AI_ACTION`

| Key | Type | Range | Description | Default |
| - | - | - | - | - |
| alt_battle | int | [0, inf) | The alternative battle animation variance. 0 means default. All variances are defined inside `/Data/AnimationSheets/anim_arkdoll.json`| 0 |
| alt_idle | int | [0, inf) | The alternative idle animation variance. 0 means default. All variances are defined inside `/Data/AnimationSheets/anim_arkdoll.json`| 0 |
| alt_move | int | [0, inf) | The alternative move animation variance. 0 means default. All variances are defined inside `/Data/AnimationSheets/anim_arkdoll.json`| 0 |
| anims | list\<`AI_ANIM`> | - | The animations that will be performed. | [] |
| at_acc_rate | float | Any | The change to `AT` per second during this action. | 0 |
| at_offset | float | Any | The change in `AT` after performing this action. | 0 |
| at_reset | bool | {true, false} | Whether this action resets `AT` to `at_base`. | false |
| busy | bool | {true, false} | Whether the AI is in `Busy` state while performing the action. | false |
| busy_time | list\<float> | [0, inf) | The minimum time in seconds that the AI is in `Busy` state while performing the action. **[1]** | [0] |
| category | int | {0, 1, 2} | The category of the action, 0 = `Defensive`, 1 = `Offensive`, 2 = `Reactive`. | 0 |
| cd | float | [0, inf) | The cool down time in seconds of this action. | 0 |
| cd_init | float | [0, inf) | The time in seconds since the AI spawns before this action can be performed. | 0 |
| charge_offset | int | Any | The change in `charge` after performing this action. | 0 |
| charge_req | int | [0, inf) | The minimum `charge` level required to perform this action. | 0 |
| dist_req | list\<float> | [0, inf) | The range of distance required between the AI and the target to perform this action. **[2]** | [] |
| dist_req_force | bool | {true, false} | Whether AI satisfies `dist_req` before plays animation. **[2]** | false |
| events | List\<List\<`string`>> | - | Set of events that this action depends on. **[3]** | [] |
| flank | bool | {true, false} | Whether AI should flank. Note that enable this will also enable `ticket`. | false |
| follow | list\<float> | [0, inf) | The range of distance AI will keep with the target. **[4]** | [] |
| hp_req | list\<float> | [0, 1] | The HP% of the AI required to perform this action. **[5]** | [0] |
| hp_target_req | list\<float> | [0, 1] | The HP% of the target required to perform this action. **[6]** | [0] |
| interruptable | bool | {true, false} | Whether this action can be interrupted by `Reactive` action in `Busy` state. | false |
| look_at_target | bool | {true, false} | Whether AI should look at the target during the action. | false |
| max_cnt | int | [0, inf) | The maximum amount of times this action can be performed. 0 means infinity. | 0 |
| move_spd_scale | float | [0, inf) | The scale to move speed during this action. | 1 |
| priority | int | Any | The priority of the action. Larger value means higher priority. **[7]** | 0 |
| probability | float | [0, 1] | The probability that this action will be performed if its condition is satisfied. | 1 |
| target_req | bool | {true, false} | Whether this action requires a target. | true |
| ticket | bool | {true, false} | Whether this action uses ticketing system. | true |

Notes:

**[1]** If the list has two values, the effective busy time of the action will be a random number in the range.

**[2]** If the list has one value, it represents the minimum distance between the AI and the target required for this action. If the list has two values, they represent a range of distance between the AI and the target required for this action. If `dist_req_force` is set to true, then `dist_req` is no longer treated as a condition of the action, but a prerequisite before AI performs the animation. In other words, AI will try to satisfy `dist_req` before play the animation if `dist_req_force` is true. If `dist_req_force` is set to true, additionally `busy_time` will be used as the maximum time that AI will try to satisfy `dist_req`.

**[3]** The inner list represents a set of events that are AND'd together. The outer list represents the OR of the inner lists. All the available event names are defined in [Event System](#event-system). Note that this configuration also supports adding a `!` in front of the event name to represent requiring the event not set. E.g. "in_enemy_atk_range" represents the action requires event `in_enemy_atk_range` to be set, while "!in_enemy_atk_range" represents the action requires event `in_enemy_atk_range` to be not set.

**[4]** Note that this is only applicable when the action has no animation. If the list is empty, then there is no follow distance. If the list has one value, it represents the minimum distance that the AI will try to keep with the target. If the list has two values, they represents the minimum and maximum distance that the AI will keep with the target.

**[5]** If the list has one value, it represents the minimum HP% required for this action. If the list has two values, they represent a range of the HP% required for this action.

**[6]** If the list has one value, it represents the minimum target HP% required for this action. If the list has two values, they represent a range of the target HP% required for this action.

**[7]** This parameter is used to pick an action when multiple actions' conditions are satisfied. Refer to [Action Priority](#action-priority) section for more information.

---

## AI Algorithm

0. Determine whether AI can make a decision:

    (a). If the AI cannot make a decision, [exit]. Following condition can make this happen:

        - AI character is not controllable (dead, stunned, etc).

    (b). Otherwise, proceed to #1.

1. Determine whether AI is currently in `Busy` state:

    (a). If the AI is in `Free` state, proceed to #2.

    (b). Determine whether the current action is still `Busy`. If so, determine if the current action can be interrupted, if not interruptable, [exit].

    (c). If the current action is no longer `Busy`, mark the AI as in `Free` state now and continue to #2. Following condition can make this happen:

        - Current action has finished or otherwise interrupted.
        - AI is interrupted.
        - This action is not a busy action and the busy_time has run out.
        - Current action has a series of animations (`anims`) but the target is no longer valid in between.

    (d). Otherwise, if the AI does not have any `Reactive` action, [exit].

    (e). Otherwise, the AI is still in `Busy` state and proceed to #2.

2. Determine if AI needs to re-evaluate the current state of the world. AI will re-evaluate if one the following is true:

        - If the AI has never evaluated before.
        - If the time gap to the last evaluation exceeds eval_rate.
        - If the state of the world has changed between now and the last evaluation.
    
    Proceed to #3 afterwards.

3. AI has a reasonably fresh evaluation of the world. AI will determine the current best target from the evaluation. Proceed to #4.

4. Generate the set of events that is currently happening as described in [Event System](#event-system). Proceed to #5.

5. Loop through all the `Reactive` actions and add all the ones that have their conditions satisfied to the available action list.

    (a). If there is no valid `Reactive` action found and if the AI is in `Busy` state, [exit]

    (a). If there is no valid `Reactive` action found and if the AI is not in `Busy` state, proceed to #6.

    (b). Otherwise, proceed to #9.

6. Compute the current `AT` according to the formula described in [Aggressive Tendency](#aggressive-tendency) and determine whether to take an `Offensive` or `Defensive` action.

    (a). If decided to take an `Offensive` action, proceed to #7.

    (b). If decided to take a `Defensive` action, proceed to #8.

7. AI has decided to take an `Offensive` action. Loop through all available `Offensive` actions and add all the ones that have their conditions satisfied to the available action list.

    (a). If there is no valid `Offensive` action found, proceed to #8.

    (b). Otherwise, proceed to #9.

8. AI has decided to take a `Defensive` action. Loop through all available `Defensive` actions and add all the ones that have their conditions satisfied to the available action list. Proceed to #9.

9. AI has a list of possible actions. AI then picks the action with highest priority from the list according to the algorithm described in [Action Priority](#action-priority). Note that if multiple actions tie on the priority, a random one with be chosen. 

    (a). If no valid action is found, AI will default to `Idle` action, [exit].
    
    (b). Otherwise, proceed to #10.

10. AI will perform the selected action, [exit].

---

## AI Configuration

AI uses a configuration file (located under `/Data/AI` folder) to define the parameters used for computing `AT` and define the set of available actions. Table below defines the details of this
configuration.

`AI_CONFIGURATION`

| Key | Type | Range | Description | Default |
| - | - | - | - | - |
actions | Dict\<string, List\<`AI_ACTION`>> | - | List of [Action Configurations](#action-configuration) by `Weapon` category. **[1]** | {} |
at_base_high | float | Any | Base `AT` value if HP% >= `at_base_threshold`. | 1 |
at_base_low | float | Any | Base `AT` value if HP% < `at_base_threshold`. | 0 |
at_hp_threshold | float | [0,1] | The HP% threshold for the AI to pick between `at_base_high` or `at_base_low` as the base `AT` value. | 0 |
at_threshold | float | Any | If `AT` >= `at_threshold`, AI picks an `Offensive` action, otherwise, AI picks a `Defensive` action. | 0 |
atk_range_hint | float | [0, inf) | A hint of the AI's attack range. | 0 |
eval_rate | float | [0, inf) | The minimum time in seconds that the AI will re-evaluate the state. 0 means whenever possible. | 5 |
vision | float | [0, inf) | The maximum distance between the AI and the target that will be considered. | 10 |

Note:

**[1]** All the supported `Weapon` categories are defined below, refer to `/Data/Actions` for all the available actions for each `Weapon` categories:

- none

    This is the default "no-weapon" category. Note that no matter what weapon AI character currently has, actions listed under this category is always considered.

- shw

    This is the "single-hand-weapon" category. Refer to `/Data/Actions/act_shw_*.json`.

- dw

    This is the 'dual-wield-weapon' category. Refer to `/Data/Actions/act_dw_*.json`.

---

## AI Examples

1. To implement the simplest AI, that only knows to attack its target, we can configure the `AT_Base` of the AI to be 1 all the time, and only implement an `Attack` action that does not modify `AT`.

2. To implement a `Follow` action, that the AI will try to keep a distance to the target, we can simply give AI a `Defensive` action with a `follow` parameter. Without any animation (empty `anims`), AI will just by default keep the distance to the target. We can also change the `alt_move` and `alt_battle` parameter to have the AI play a different animation. In addition, we can set `look_at_target` to true so the AI will always face the target while moving.

3. To implement a `Flank` action, that is a slightly advanced `Follow` as the AI will try to move around the target, we can simply adapt the `Follow` action by setting `flank` parameter to true.

4. To implement a mob like AI, that has a `hesitate -> attack -> hesitate` loop, we can configure the `AT_Base` of the AI to be 0 initially, and give AI some `Attack` actions that decrements `AT` along with a `Follow`/`Flank` action. We also need to set `at_acc_rate` of the `Defensive` actions to be a positive value. This way, AI will first perform the `Follow`/`Flank` action until it accumulates enough `AT`. Once reached the critical point, AI will then perform `Attack` actions that consume the `AT` and naturally fallback to being defensive.

5. To implement an AI that has a `Dash&Attack`, we can configure the AI with an `Attack` action that has a `dist_req`, and use one of the dash-attack animations. We should also configure this action with a higher `priority` than other normal `Attack` actions. This way when AI turns to `Offensive` and find it far from its target, a `Dash&Attack` can be performed instead of the normal `Attack`.

6. To implement an AI that has an `Ultimate`, we can configure the AI with an `Attack` action that has some `cd_init`, and give this action a higher `priority` than other normal `Attack` actions. This way, when the initial cool down of this action is finished, the AI will favor this action over the others. This approach ensures the AI will use the `Ultimate` at a roughly uniform cadence.

7. Another way to implement `Ultimate` for the AI is to use the `charge` system. We can define a set of normal `Attack` or even other actions that increments the `charge`. And then define the `Ultimate` to be a higher `priority` `Attack` action that consumes a certain level of `charge`. This way, the AI will need to perform the normal actions to build up the `charge` before it can use the `Ultimate`. This approach ensures the AI will perform roughly the same prologue before the finale.

8. To implement a `Rest` action, that the AI will perform some resting animation that can be used by the player as a weak point, [TODO]

9. To implement a `Intimidate` action that the AI will perform some preparing animation before attack that can be used by the player as a hint, [TODO]

10. To implement a `Run&Hit` action, that the AI will first keep a distance to the target and then perform an attack, we can add an `Attack` action that has a `dist_req` and also set `dist_req_force` to true. This way AI will be always attempt to satisfy `dist_req` before perform the attack animation. We can further configure a `busy_time` to make sure AI does not move forever.

11. To implement a `Flee` action, that the AI will keep moving to the lowest threat position, [TODO]

12. To implement a `Counter` action, that the AI will react to the target's attacks with a counter-attack or block animation, we can achieve this with two actions. First we can create a `Follow` action and have the AI play an alternative move/battle animation. And also give AI a `Reactive` action that listens for `in_enemy_active_atk_range` or `in_target_active_atk_range` event, and this action will perform the counter animation.

13. To implement an AI that can `Fake`, such that the AI can perform a dummy attack animation and immediately turn to `Counter` if the target attacks, or turn to `Attack` otherwise, we can achieve this with two actions. First we can create an `interruptable` `Fake` animation that performs the first half of the attack, note that this animation will not actually trigger the attack. And during this action we will increment `AT`. Also, we add a `Counter` action. Finally, we add an `Attack` action that continues from the fake attack, and gate this `Attack` action behind the `AT` accumulated by `Fake`. This way, if `Counter` fires during the `Fake`, AI will counter. If the `Fake` plays to the end, AI will turn to `Attack`.