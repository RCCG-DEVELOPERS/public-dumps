# Who can do what — hierarchy, transfers and principal officers

A plain-language guide for administrators. It says who may perform each
operation and who, if anyone, has to approve it. The technical copy with
endpoints is `authority_matrix_for_hierarchy_and_unit_transfers.md`.

Current as of 18 September 2026. Everything on this page is live.

## The terms used below

| Term | Meaning |
|---|---|
| **Super Admin / National Support** | May do everything on this page, anywhere, with no approval. |
| **Administrator** | Province Admin, Regional Admin, Sub-Continent Admin, Continent Admin. Assistants and the Pastor in Charge of a province are **not** administrators. |
| **Your unit** | The province, region, area and so on that your role sits in, as recorded on your own profile. |
| **Contains** | A parish is *contained* by its area, its zone, its province, its region and so on up the chain. |
| **Within your unit** | Both the thing being moved and where it is going sit inside your unit. |

---

## 1. Moving parishes, areas and zones

| What you want to do | Who can do it | Who must approve |
|---|---|---|
| Move a parish, area or zone to a new parent **within your own unit** | Anyone with a role over a unit that holds both the old and the new location — a Province Admin or Pastor in Charge within their province, a Regional Admin within their region | Nobody |
| **Receive** a parish, area or zone from another province into yours | The **Administrator** of the receiving side — the Province Admin of the province it is joining, or the Regional Admin of a region that holds it | Nobody. Your action is the approval. If the giving province had already asked for it to come to you, that request is closed as approved. If they asked for it to go somewhere else, you are told, and that other province decides. |
| The same, but you are an assistant or the Pastor in Charge of the province | Not allowed. You are told which administrator can | — |
| **Send** a parish, area or zone out to another province | The Province Admin of the province it is leaving, or an administrator above them | The Province Admin of the province it is **joining**; or the Regional Admin whose region holds that province; or Super Admin. Never the person who asked. |
| Move anything, anywhere, including whole provinces | Super Admin / National Support | Nobody |
| Move a unit's **headquarters parish** out from under it | Super Admin / National Support only. Everyone else must first name a different headquarters, then move the parish | Nobody |
| Repair a unit whose parishes disagree about which province or region they are in — see [section 9](#9-realigning--repairing-a-unit-that-has-come-apart) | Super Admin / National Support anywhere; an Administrator when both sides of the disagreement are inside their own unit | Nobody |
| Check in advance what a move would do and whether you are allowed | Anyone signed in | — |
| Undo a completed move | Super Admin / National Support | Nobody |

## 2. Promoting a unit — parish to area, area to zone, and so on

| What you want to do | Who can do it | Who must approve |
|---|---|---|
| Promote a unit inside your own unit | An Administrator whose unit holds it | Nobody |
| Promote a unit and fold other units into it at the same time | Super Admin / National Support | Nobody |
| A promotion that would demote a headquarters parish | Super Admin / National Support | Nobody |

## 3. Principal officers — appointing to the register

| What you want to do | Who can do it | Who must approve |
|---|---|---|
| Appoint an officer at **your own** unit — a Province Admin appointing the Pastor in Charge of their province | Anyone with a role at that unit | Nobody |
| Appoint an officer in a unit **inside yours** — an Area Admin appointing a parish pastor in their area; a Province Admin appointing anywhere in their province | The Administrator of the containing unit | Nobody |
| Appoint an officer **above** you — an Area Admin appointing a province officer | Not allowed. Raise a request instead (section 4) | — |
| Appoint someone into a unit that is not their home unit | As above, and the person must already hold that role there as a secondary grant | Nobody |
| Appoint a Training Manager | As above. Training Managers hold no administrative authority | Nobody |
| End an appointment | Super Admin / National Support | Nobody |
| Hand an appointment to another person | Super Admin / National Support | Nobody |

## 4. Principal officers — by request and approval

| What you want to do | Who can ask | Who approves |
|---|---|---|
| Replace the holder of an office | Anyone signed in | An Administrator whose unit holds the office, at the same level or above; or Super Admin. Never the person who asked. |
| Move a person into a different office | Anyone signed in | Same as above |
| Withdraw a request | The person who raised it, or Super Admin | — |

## 5. Principal officers — keeping the register clean

| What you want to do | Who can do it |
|---|---|
| See people holding an officer role with no appointment, and appointments with no role | Super Admin / National Support |
| Remove an officer role from people who hold no appointment | Super Admin / National Support. Refused for anyone who is genuinely appointed |
| See offices with two holders, or with none | Super Admin / National Support |
| See the officers and vacancies in your own units | Anyone with a role; you see your own units only |

## 6. Pastor in charge of a parish

This is the record of who actually leads a parish. Holding the parish pastor
role is not the same thing: many people hold the role who lead no parish.

| What you want to do | Who can do it | Who must approve |
|---|---|---|
| Appoint the pastor in charge of a parish | The Administrator of any unit that holds the parish — Area Admin, Zone Admin, Province Admin, Regional Admin and above | Nobody |
| A parish pastor appointing their own successor | Not allowed. A parish cannot appoint its own head | — |
| End a pastor's appointment | Same as appointing | Nobody |
| See who leads a parish | Anyone signed in | — |

## 7. People and headquarters

| What you want to do | Who can do it | Who must approve |
|---|---|---|
| Move a member to another parish | Anyone signed in may ask | The Province Admin of the province they are leaving **or** joining; or Super Admin. Never the person who asked. |
| Name, remove or fix a unit's headquarters parish | Anyone with a role over that unit | Nobody |
| Give someone a role | Only roles you are permitted to grant; the system tells you which | Nobody |

## 8. Requests in general

| Action | Who |
|---|---|
| Approve or reject | The people named in the tables above. Nobody approves their own request, except Super Admin. |
| Withdraw | The person who raised it, or Super Admin |
| See requests | Those in your own units; Super Admin sees all |

---

## 9. Realigning — repairing a unit that has come apart

### What has gone wrong

A province does not carry a record of which region it belongs to. Its parishes
do, and so do its members. So a province is *split* when its own parishes
disagree — half of them saying they are in one region, half saying another.

This usually happens when a headquarters parish is moved away. A parish is the
smallest unit, so moving it changes that parish and nothing else. If it happened
to be the headquarters of a province, everything else in that province stays
where it was, and the province is now claimed by two regions at once.

You will notice it because the unit stops working. Any attempt to move,
preview or promote it is refused, with a message saying the unit is
inconsistent. Realigning is the only way to clear that.

### What realigning does

It puts every parish in the unit, and every member of it, under one parent —
the one you choose. Nothing moves. The unit stays exactly where it is; you are
correcting the record of a move that already happened.

| It does | It does not |
|---|---|
| Update every parish in the unit to name the same parents | Move the unit anywhere |
| Update every member of those parishes to match | Change the unit's own name or code |
| Include departments, which an ordinary move leaves alone | End anyone's appointment as an officer |
| Keep a record of every row it changed, so it can be undone | Decide for you. Where the parishes disagree, you pick which side is right |

Officers are deliberately left alone. Moving a unit ends the appointments held
at the unit it left, but realigning is a correction, not a move, and ending
someone's appointment as a side effect of tidying data would be both destructive
and a guess. Any appointments worth reviewing are listed for you, and you decide
what to do with each.

### Which unit goes under which

You always name the level **immediately above** the unit. Everything higher is
worked out from there.

| Realign this | Under this | What usually caused it |
|---|---|---|
| A parish | its area | Duplicate records for the same parish that disagree |
| An area | its zone | The area's parishes name two different zones or provinces |
| A zone | its province | The zone's parishes name two different provinces |
| A province | its region | The province headquarters was moved to another region and the rest of the province stayed behind. This is the common one |
| A region | its sub-continent | A province moved between regions and left inconsistent spellings behind |
| A sub-continent | its continent | Rare, and usually only a spelling difference |
| A continent | — | Not possible. A continent has nothing above it |

### Who can do it

| Who | What they can repair |
|---|---|
| Super Admin / National Support | Any unit, anywhere, however far the split reaches |
| An Administrator | A unit inside their own, **provided both sides of the split are also inside their own unit** |

That second condition matters more than it looks. A split unit is, by
definition, partly somewhere else. If that somewhere is outside your unit, then
repairing it would mean pulling part of another administrator's territory into
yours on your own say-so, which is exactly the decision that needs Super Admin.

| Realign this | You need to be at least | Example |
|---|---|---|
| A parish | Area Admin | An Area Admin tidies duplicate records of a parish in their area |
| An area | Zone Admin | A Province Admin repairs an area whose parishes name two zones **within their province** |
| A zone | Province Admin | A Province Admin repairs a zone that drifted inside their province |
| A province | Regional Admin | A Regional Admin repairs a province split between two of **their own** regions |
| A region | Sub-Continent Admin | Rare, and normally Super Admin work |

So a Province Admin whose province has drifted into **another** region cannot
repair it themselves. They are told so, and told that Super Admin or National
Support can.

### When to use it

- **A headquarters parish was moved out**, and the unit it used to head is now
  claimed by two parents. This is what the feature was built for.
- **A unit has stopped working** — moves, previews and promotions on it are all
  being refused as inconsistent.
- **Only the spelling differs.** Every record agrees on which unit it belongs
  to, and differs only in how the name is written, such as `REGION 71` against
  `Region 71`. Harmless, and cleared in one go.
- **A large change stopped part-way through**, leaving some records updated and
  others not.
- **Older or imported records** where parishes were edited one at a time.

Do **not** use it to move a unit somewhere new. If a unit genuinely belongs
elsewhere, that is a transfer. If you try to realign a unit that is already
consistent, you are told there is nothing to repair.

### Doing it safely

Always ask for a preview first. Which side of the disagreement wins **cannot be
worked out again afterwards**, so it is worth reading before you commit.

The preview tells you:

- every version of the ancestry the unit's parishes currently claim, largest
  group first, so you can see which is the majority
- whether the destination is itself split, and which of its versions will be
  used
- how many parishes and how many people would be updated
- any headquarters this would strand, which stops the operation unless Super
  Admin explicitly accepts it
- which officers hold appointments at a unit this one will no longer sit under,
  so you can review them afterwards
- a ready-made confirmation to send with the real request

Send that confirmation back with the real request. It carries proof of what you
were shown, so if the data changes in between, the operation stops rather than
quietly writing a different answer from the one you approved.

### Afterwards

Two things are worth doing. Review any appointments that were flagged, and end
or move them if they no longer make sense. Then check the unit again and confirm
it no longer shows as split. If the repair was wrong, it can be undone — every
record changed was kept.

---

## Recently changed, all live

- A Regional Admin may approve a move between two provinces in their own region.
- The receiving province's Administrator may accept a parish, area or zone from another province on their own.
- An Administrator may appoint principal officers in the units below them.

## One thing to decide

Ending or handing over a principal-officer appointment is still Super Admin
only, while appointing is delegated. An Area Admin can appoint a parish pastor
to the register but cannot end that appointment. Say the word and
the two will follow the same rule.
