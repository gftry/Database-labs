# Gym Logical Database Design

## Overview

DBAD 4000 - Class Activity: Designing a Logical Database.

This logical model supports gym membership, trainers, scheduled class sessions, class bookings and completed payments. It contains six entities and 26 attributes, including `Enrollment.is_cancelled`.

**Team:** Troy Franks, Seth Garciano, Tristin Bates and Yisong Wang.

## Project files

| File | Contents |
| --- | --- |
| [Data_Dictionary_and_Team_Contribution.pdf](Data_Dictionary_and_Team_Contribution.pdf) | Complete data dictionary, validation constraints, foreign-key references and recorded team contributions |
| [gym-eerd.drawio](gym-eerd.drawio) | Editable EERD in diagrams.net format |
| [README.md](README.md) | Business rules, model design, relationships, normalization and design rationale |

## Scope and assumptions

This proposed logical model supports one gym location and six entities: Person, Member, Trainer, ClassSession, Enrollment and Payment. It is aligned with [gym-eerd.drawio](gym-eerd.drawio). These are design assumptions for the activity rather than facts collected from a real gym.

All payments use CAD. Timestamps represent gym local time in America/Edmonton. Membership plans, recurring class templates, room allocation, payroll, invoices, refunds, unsuccessful or pending payment attempts and multiple trainer certifications are outside the current scope. Enrollment records retain the current cancellation state; a complete history of cancellation and re-enrollment events is outside scope.

## Business rules

### BR1 People and roles

Each Person has one unique person_id and one unique email address. Every stored Person must be a Member, a Trainer, or both. This is a total, overlapping specialization: one individual can hold both roles while sharing the same name and contact details.

### BR2 Membership

Each Member belongs to exactly one Person and uses that Person's person_id as both its primary key and foreign key. A membership has a join date and a status of active or inactive. An active Member may have no class enrollments. Only active members may create new enrollments or restore cancelled enrollments. Historical enrollment and payment records are retained when a membership becomes inactive. Changing membership_status does not automatically change Enrollment.is_cancelled; cancelling a class booking is a separate action and does not change membership_status.

### BR3 Scheduled classes

Each ClassSession represents one scheduled occurrence with a class name, start date and time, positive duration and positive capacity. Exactly one Trainer leads each session. A Trainer may lead zero or many sessions. Repeated occurrences of the same class name have different class_id values.

### BR4 Class enrollment

A Member may enroll in zero or many ClassSessions, and each ClassSession may have zero or many members. Enrollment resolves this M-M relationship. Each Enrollment belongs to exactly one Member and one ClassSession. The composite primary key (member_id, class_id) permits only one record for each member-session pair, including cancelled records.

Enrollment.is_cancelled is a required BOOLEAN with a default of FALSE. FALSE means the booking has not been cancelled and counts towards the session capacity. Cancelling a booking sets is_cancelled to TRUE, keeps the Enrollment row and releases its place. The number of Enrollment rows with is_cancelled = FALSE for a session must not exceed ClassSession.capacity; cancelled records do not count towards this limit.

To re-enroll in the same session after cancellation, restore the existing row by setting is_cancelled to FALSE after checking that the member is active and a place is available. Do not insert a second row for the same pair. enrolled_at retains the original creation timestamp; restoring an enrollment does not overwrite it. This model records the current cancellation state rather than every cancellation and re-enrollment event.

### BR5 Completed payments

Each Payment records one completed, positive payment from exactly one Member. A Member may have zero or many payments. All amounts are received amounts in CAD, paid_at is required and records completion time, and payment_method must be cash, debit or credit. Pending, failed and cancelled payment attempts are not stored. Payment therefore has no payment_status attribute. Payments are independent of class enrollment; a payment does not create or identify a class booking, and cancelling a booking does not remove a completed payment.

### BR6 Trainer details

Each Trainer belongs to exactly one Person and uses that Person's person_id as both its primary key and foreign key. The model records one primary certification and one hire date per Trainer. Trainer-specific details are stored separately from shared personal details and membership details.

## Design overview

| Entity | Purpose | Primary key |
| --- | --- | --- |
| Person | Shared identity and contact information | `person_id` |
| Member | Membership details and eligibility to book classes | `person_id`, also FK to Person |
| Trainer | Trainer-specific certification and employment details | `person_id`, also FK to Person |
| ClassSession | One scheduled class occurrence led by one trainer | `class_id` |
| Enrollment | A member's booking and current cancellation state for a session | Composite `(member_id, class_id)` |
| Payment | One completed payment received from a member | `payment_id` |

### Specialization and shared identity

Person is the supertype; Member and Trainer are subtypes. Specialization is **total**: every Person must have at least one of these roles. It is **overlapping**: one Person may be both a Member and a Trainer. Sharing the Person primary key gives each subtype exactly one parent without duplicating contact information. The `is a` branches express this inheritance; they do not mean that every Person must have both roles.

### Enrollment and booking cancellation

Enrollment resolves the Member-ClassSession M-M relationship into two 1-M relationships. Its identity depends on the member-session pair, and the composite key prevents duplicate rows for that pair. `is_cancelled` is a required BOOLEAN with a default of FALSE. Cancellation preserves the row while releasing its place; capacity checks count only rows with `is_cancelled = FALSE`.

Re-enrollment restores the existing row after checking membership eligibility and available capacity. `enrolled_at` remains the original row-creation time. A separate event log would be needed to retain every cancellation and restoration, which is outside this activity's scope.

### Membership and payment boundaries

An active Member can use the gym without enrolling in a class. A session requires a trainer, but a Member does not need an assigned trainer. Membership state and booking cancellation describe different facts: changing one does not automatically change the other.

Payment stores completed transactions only. `paid_at` is required, `amount` is the positive amount received in CAD, and `payment_method` is restricted to cash, debit or credit. Pending, failed and cancelled attempts are excluded, so no payment_status or attempt-creation field is needed. A completed payment does not itself create a class booking.

## Relationships and participation

| Relationship | Type | Participation |
| --- | --- | --- |
| Person to Member | ISA; 1-1 shared-key mapping | 1 Person per Member; 0..1 Member role per Person |
| Person to Trainer | ISA; 1-1 shared-key mapping | 1 Person per Trainer; 0..1 Trainer role per Person |
| Trainer to ClassSession | 1-M | 1 Trainer per session; 0..M sessions per Trainer |
| Member to Enrollment | 1-M | 1 Member per enrollment; 0..M enrollments per Member |
| ClassSession to Enrollment | 1-M | 1 session per enrollment; 0..M enrollments per session |
| Member to Payment | 1-M | 1 Member per payment; 0..M payments per Member |
| Member to ClassSession | M-M at the business level | 0..M on both sides; implemented through Enrollment |

M means many. A 1-M label gives the maximum relationship type; 0..M explicitly permits zero related records. Each subtype is individually optional, but the total specialization requires every Person to belong to at least one subtype. Overlapping specialization allows a Person to belong to both.

The ISA branches express subtype inheritance. The 1-1 correspondence describes the shared primary keys; it does not require every Person to have both roles. Enrollment implements the Member-ClassSession M-M association as two 1-M relationships, so no additional direct relationship is required in the logical model.

Enrollment cardinalities describe stored records, including cancelled rows. Current bookings are the subset with is_cancelled = FALSE; cancellation does not change the structural relationships or their participation rules.

## Normalization and design rationale

- **1NF:** Each field holds one value. Class selections are separate Enrollment rows rather than a list inside Member. Only one primary certification is recorded per Trainer.
- **2NF:** Enrollment uses the whole pair (member_id, class_id) as its primary key. Its non-key attributes, enrolled_at and is_cancelled, describe that specific enrollment and depend on both IDs. All other entities have single-attribute primary keys.
- **3NF:** Shared contact details appear only in Person. ClassSession stores trainer_id rather than trainer details, and Payment stores member_id rather than member details. Under these assumptions, no non-key attribute determines another non-key attribute.
- **Scope:** class_name is a session label, not a unique catalogue key; it does not determine duration, capacity or trainer. A payment amount is the recorded transaction amount, not a price determined by payment_method.

### Other design decisions

- **Scheduled occurrences:** ClassSession identifies a particular scheduled session. Repeated Yoga sessions have different class_id values; class_name is not a catalogue key.
- **Role-specific foreign keys:** ClassSession.trainer_id references Trainer, while Enrollment.member_id and Payment.member_id reference Member. This ensures the referenced person holds the required role.
- **No stored booking count:** The number of non-cancelled bookings is calculated from Enrollment. Storing another count would create a second value that must be synchronized after every cancellation or restoration.
- **History retention:** Membership inactivation and booking cancellation preserve existing Enrollment and Payment records.
- **Bounded trainer details:** Only one primary certification is recorded per trainer; a separate collection of certifications is outside scope.

## Integrity and validation

All fields are required except Person.phone. All identifiers are positive integers. Duration, capacity and payment amount must be positive. Email values are unique after trimming and lowercase normalization. The data dictionary defines the field lengths, allowed values, key references and nullability.

The total specialization requires checking Person together with Member and Trainer. Creating or restoring an enrollment must check membership_status at the time of booking. Capacity checks count only Enrollment rows where is_cancelled = FALSE. Check capacity atomically when a booking is created or restored, or when a session capacity is reduced, so simultaneous bookings cannot overfill a class. A membership-status change alone does not release any reserved places; the corresponding bookings must be cancelled separately to release them. Ordinary foreign keys alone do not enforce these rules. This activity specifies the logical model; it does not claim that a database implementation already exists.

## Design process and tools

The model was organized around the business rules, then refined by separating shared Person information from role-specific data, assigning keys, resolving the class-enrollment M-M relationship, specifying participation and reviewing functional dependencies through 3NF. The current revision adds cancellation tracking and keeps Payment limited to completed transactions. Field names and key mappings were checked for consistency across the EERD and data dictionary.

The EERD is editable in diagrams.net (draw.io). Documentation is maintained in Markdown, and the data dictionary and team contribution PDF was generated with Python and ReportLab. The deliverables describe a logical design; they do not include an implemented database or SQL enforcement.

## Team contributions

The four members' recorded contributions are included in the companion [data dictionary and team contribution PDF](Data_Dictionary_and_Team_Contribution.pdf), using the contribution records supplied by the team.

## Source and AI assistance

[SAIT Brightspace: Class Activity - Designing a Logical Database](https://learn.sait.ca/d2l/le/lessons/886971/topics/21946635).

OpenAI Codex assisted with drafting and revising the EERD, business rules, data dictionary, normalization explanations and document formatting. The contribution table reproduces the team's supplied records; it does not assign additional work to individual members. This disclosure identifies the assistance used in preparing these materials.
