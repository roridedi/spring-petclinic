# PRD: Appointment Scheduling

## 1. Introduction / Overview

Pet owners currently have no way to schedule a vet visit online — they must call the clinic. This feature adds a self-service **appointment scheduling** system to Spring PetClinic so owners can book a future vet visit for one of their pets, pick a vet and a time slot, and receive an email confirmation. Clinic staff can also create, edit, reschedule, and cancel appointments on behalf of owners.

A new `Appointment` entity represents *scheduled future events*. The existing `Visit` entity (`src/main/java/org/springframework/samples/petclinic/owner/Visit.java`) remains the record of a *completed* visit and is unchanged by this feature. When an appointment is fulfilled, staff record the outcome as a `Visit` (out of scope for this PRD — flagged in Open Questions).

## 2. Goals

- Allow any owner to self-book a vet appointment online without phone contact.
- Allow staff to create, reschedule, and cancel appointments on behalf of any owner.
- Prevent double-booking the same vet at the same time slot.
- Send an email confirmation when an appointment is booked, and a reminder before it occurs.
- Reduce the share of bookings made by phone (measurable post-launch — see §8).

## 3. User Stories

Each story is sized for one focused implementation session.

### US-001: Add `Appointment` entity and repository
**Description:** As a developer, I need an `Appointment` domain entity persisted to the database so future appointments can be stored independently of historical `Visit` records.

**Acceptance Criteria:**
- [ ] New entity `org.springframework.samples.petclinic.owner.Appointment` extends `BaseEntity` with fields: `pet` (`@ManyToOne` → `Pet`), `vet` (`@ManyToOne` → `Vet`), `startTime` (`LocalDateTime`, not null), `durationMinutes` (`int`, default 30), `reason` (`String`, optional, max 255), `status` (enum: `SCHEDULED`, `CANCELLED`, `COMPLETED`; default `SCHEDULED`), `createdAt` (`LocalDateTime`).
- [ ] Bean Validation: `startTime` must be in the future on create (`@Future`); `reason` length ≤ 255.
- [ ] `AppointmentRepository extends Repository<Appointment, Integer>` with: `findByPetId(int)`, `findByVetIdAndStartTimeBetween(int, LocalDateTime, LocalDateTime)`, `findByStartTimeBetween(LocalDateTime, LocalDateTime)`, `save`, `findById`, `delete`.
- [ ] Schema updates in all three DB scripts: `src/main/resources/db/h2/schema.sql`, `db/mysql/schema.sql`, `db/postgres/schema.sql` — create `appointments` table with FK to `pets(id)` and `vets(id)`, plus index on `(vet_id, start_time)`.
- [ ] `mvn -q test` and `./gradlew test` pass.

### US-002: Slot computation service
**Description:** As a developer, I need a service that returns the available appointment slots for a given vet and date, so the UI can show only bookable times.

**Acceptance Criteria:**
- [ ] New `AppointmentSlotService` in `org.springframework.samples.petclinic.owner` with method `List<LocalDateTime> availableSlots(int vetId, LocalDate date)`.
- [ ] Working hours hardcoded as constants: Mon–Fri, 09:00–17:00, 30-minute slots (FR-9). Weekends return an empty list.
- [ ] Excludes slots already taken by a `SCHEDULED` appointment for that vet (uses `findByVetIdAndStartTimeBetween`).
- [ ] Excludes slots in the past (if the requested date is today, only future slots are returned).
- [ ] Unit test covers: weekday with no bookings (returns 16 slots), weekday with one booking (returns 15), Saturday (returns 0), today with current time after 14:00 (returns only slots ≥ next half-hour).
- [ ] Typecheck/build passes.

### US-003: Booking form — pick vet, date, time
**Description:** As an owner, I want to book a future appointment for my pet by selecting a vet, a date, and an available time slot, so I don't have to call the clinic.

**Acceptance Criteria:**
- [ ] New `AppointmentController` exposes:
  - `GET /owners/{ownerId}/pets/{petId}/appointments/new` — renders booking form (vet dropdown, date picker, slot dropdown, optional reason textarea).
  - `POST /owners/{ownerId}/pets/{petId}/appointments/new` — validates and persists; redirects to `/owners/{ownerId}` with flash message `"Your appointment is confirmed for <date> <time> with Dr. <vet>"`.
- [ ] Date picker defaults to tomorrow; only weekdays selectable.
- [ ] Slot dropdown is populated via an AJAX call to `GET /api/appointments/slots?vetId={id}&date={yyyy-MM-dd}` returning JSON `[{"startTime":"2026-06-01T09:00:00","label":"09:00"}, ...]`. Dropdown disabled until both vet and date are chosen.
- [ ] Submitting a slot already taken (race condition) re-renders the form with error `"That slot was just taken — please pick another."` (FR-5).
- [ ] `InitBinder` disallows `id`, `*.id`, `status`, `createdAt` (mirroring the pattern in `VisitController`).
- [ ] New Thymeleaf template `src/main/resources/templates/appointments/createAppointmentForm.html` follows the `layout.html` fragment pattern and reuses `fragments/inputField.html` / `selectField.html`.
- [ ] **Verify in browser using dev-browser skill:** can complete a full booking happy path; double-booking is rejected; weekend dates are not selectable.
- [ ] Typecheck/build passes.

### US-004: Owner-facing appointment list
**Description:** As an owner, I want to see my pet's upcoming appointments on the owner details page so I know what's scheduled.

**Acceptance Criteria:**
- [ ] `OwnerController` loads upcoming `SCHEDULED` appointments for each of the owner's pets and exposes them on the model.
- [ ] `src/main/resources/templates/owners/ownerDetails.html` shows, under each pet's existing Visits table, a new "Upcoming Appointments" section with columns: date, time, vet, reason, and a "Cancel" button.
- [ ] If a pet has no upcoming appointments, the section shows `"No upcoming appointments."` and a "Book appointment" link.
- [ ] "Book appointment" link points to `GET /owners/{ownerId}/pets/{petId}/appointments/new`.
- [ ] **Verify in browser using dev-browser skill:** an owner with mixed (some pets with, some without) appointments renders correctly.
- [ ] Typecheck/build passes.

### US-005: Cancel an appointment
**Description:** As an owner or staff member, I want to cancel an upcoming appointment so the slot is freed up.

**Acceptance Criteria:**
- [ ] `POST /owners/{ownerId}/pets/{petId}/appointments/{appointmentId}/cancel` sets the appointment's `status` to `CANCELLED` and persists.
- [ ] Cancelled appointments no longer block the slot in `AppointmentSlotService` (i.e. the service filters on `status = SCHEDULED`).
- [ ] Cancelled appointments are hidden from the "Upcoming Appointments" list (US-004).
- [ ] Redirects to `/owners/{ownerId}` with flash message `"Appointment cancelled."`.
- [ ] Returns 404 if appointment does not belong to the given pet/owner.
- [ ] **Verify in browser using dev-browser skill:** cancel an appointment, confirm the slot becomes available again on the booking form.
- [ ] Typecheck/build passes.

### US-006: Reschedule an appointment
**Description:** As an owner or staff member, I want to change the date/time/vet of an upcoming appointment without cancelling and rebooking.

**Acceptance Criteria:**
- [ ] `GET /owners/{ownerId}/pets/{petId}/appointments/{appointmentId}/edit` renders the booking form pre-filled with the current vet, date, and slot.
- [ ] `POST` to the same path updates the appointment (only if `status = SCHEDULED`), enforces the same future-time and no-double-booking checks as creation, and excludes the appointment being edited from its own conflict check.
- [ ] Editing a non-`SCHEDULED` appointment returns to the owner page with flash message `"Only scheduled appointments can be edited."`.
- [ ] **Verify in browser using dev-browser skill:** edit a booking to a different vet/time and confirm both the old slot is freed and the new one is taken.
- [ ] Typecheck/build passes.

### US-007: Staff appointment console
**Description:** As clinic staff, I want a single page that lists all appointments for a given day across all vets so I can manage the schedule.

**Acceptance Criteria:**
- [ ] `GET /appointments` (with optional `?date=yyyy-MM-dd`, default today) renders a daily schedule.
- [ ] Table grouped by vet, with rows per slot; cells show pet name + owner name + reason, with links to edit/cancel.
- [ ] Date navigation: "← Previous day" / "Next day →" links.
- [ ] Linked from the top nav (`fragments/layout.html`) as a new "Appointments" menu item, visible to all users (no auth distinction exists in this codebase — see Non-Goals).
- [ ] **Verify in browser using dev-browser skill:** navigate days; click into edit/cancel actions from this page.
- [ ] Typecheck/build passes.

### US-008: Email confirmation on booking
**Description:** As an owner, I want to receive an email confirmation immediately after booking so I have a record of the appointment.

**Acceptance Criteria:**
- [ ] Add `spring-boot-starter-mail` to both `pom.xml` and `build.gradle`.
- [ ] New `AppointmentNotificationService` sends a plain-text email via `JavaMailSender` on successful create (US-003) and reschedule (US-006).
- [ ] Email subject: `"PetClinic appointment confirmed: <date> <time>"`. Body includes pet name, vet name, date/time, reason, and a cancellation link to the controller in US-005.
- [ ] Recipient is `Owner.email` — **requires adding an `email` column** to the `owners` table and entity (`@Email`, optional). Update all three `db/*/schema.sql` files and `Owner.java`.
- [ ] Existing owner edit form (`createOrUpdateOwnerForm.html`) gains an email input.
- [ ] Mail config in `application.properties` uses environment-driven values (`spring.mail.host`, `spring.mail.port`, `spring.mail.username`, `spring.mail.password`). In test profile, use a no-op or in-memory mail sender to avoid network I/O.
- [ ] If `Owner.email` is blank, sending is silently skipped and a warning is logged (no failure of the booking flow).
- [ ] Unit test verifies `JavaMailSender.send(...)` is invoked exactly once per successful booking with the correct `to` address and a non-empty body. Integration test covers the "no email on owner — skip" branch.
- [ ] **Verify in browser using dev-browser skill:** with a test SMTP catcher (e.g. MailHog) or the test profile's in-memory sender exposed via a log/endpoint, complete a booking and confirm an email was produced.
- [ ] Typecheck/build passes.

### US-009: Appointment reminder (24h before)
**Description:** As an owner, I want a reminder email roughly 24 hours before my appointment so I don't forget.

**Acceptance Criteria:**
- [ ] Scheduled job (`@Scheduled(cron = "0 0 9 * * *")`, i.e. daily at 09:00 server time) finds all `SCHEDULED` appointments where `startTime` is between 24h and 48h from now and sends a reminder email via `AppointmentNotificationService.sendReminder(...)`.
- [ ] Add `@EnableScheduling` to the application config (new `@Configuration` class or on `PetClinicApplication`).
- [ ] Add a boolean column `reminder_sent` to the `appointments` table (default `false`); the job only sends to appointments with `reminder_sent = false` and flips the flag after sending, so a job retry does not double-send.
- [ ] Subject: `"Reminder: PetClinic appointment tomorrow at <time>"`.
- [ ] Unit test uses a fixed `Clock` bean to assert the query window and the flag update. (Inject `Clock` into the service rather than calling `LocalDateTime.now()` directly.)
- [ ] Typecheck/build passes.

## 4. Functional Requirements

- **FR-1:** The system must persist appointments in a new `appointments` table separate from `visits`.
- **FR-2:** The system must allow any owner to create an appointment for any of their own pets via the owner details page.
- **FR-3:** The system must allow staff to create, edit, or cancel any appointment via the staff console (`/appointments`).
- **FR-4:** The system must reject any appointment whose `startTime` is not strictly in the future.
- **FR-5:** The system must prevent two `SCHEDULED` appointments for the same vet from overlapping in time (atomic check on save; on conflict, return the booking form with a user-visible error).
- **FR-6:** The system must compute available slots per vet per date based on (a) the working-hours configuration in FR-9 and (b) existing `SCHEDULED` appointments for that vet.
- **FR-7:** The system must expose `GET /api/appointments/slots?vetId=…&date=…` returning a JSON array of available slots for that vet/date.
- **FR-8:** The system must send a confirmation email immediately on successful create or reschedule (US-008).
- **FR-9:** Working hours are hardcoded as constants in `AppointmentSlotService`: Monday–Friday, 09:00–17:00 (local server time), 30-minute slots. Weekends are unavailable.
- **FR-10:** A daily scheduled job must send a 24h-prior reminder email exactly once per appointment (US-009).
- **FR-11:** Cancelling an appointment must free the slot (filter on `status = SCHEDULED` in slot computation).
- **FR-12:** All controllers accepting appointment form input must restrict bindable fields via `@InitBinder` (disallow `id`, `*.id`, `status`, `createdAt`, `reminderSent`).
- **FR-13:** All times are stored and computed in the server's local time zone; no per-vet or per-owner time zones in this PRD.

## 5. Non-Goals (Out of Scope)

- **Authentication / authorization.** The current PetClinic codebase has no user accounts; "owners" and "staff" are conceptual. Any owner can currently view/edit any owner's data, and that does not change here. A proper auth model is a separate effort.
- **Recurring appointments.** One-off only.
- **Per-vet working hours** stored in the DB. Hardcoded constants only (matches answer 5A).
- **Holiday / time-off** calendars for vets.
- **SMS notifications.** Email only.
- **Replacing or modifying `Visit`.** `Visit` remains the historical record. Converting a completed `Appointment` into a `Visit` is *not* covered here — see Open Questions.
- **Owner-side calendar view (ICS export, calendar embed).** Plain list only (US-004).
- **Time zone handling beyond a single server zone.**
- **Mobile-specific UI.** Standard responsive Thymeleaf pages only.

## 6. Design Considerations

- Reuse `src/main/resources/templates/fragments/layout.html`, `inputField.html`, and `selectField.html` for visual consistency with the existing booking-form pattern (`pets/createOrUpdateVisitForm.html`).
- The owner details page (`owners/ownerDetails.html`) already lists Visits per pet — place the new "Upcoming Appointments" block immediately above or below that section so the visual hierarchy is consistent.
- Staff console (US-007) should mirror the table styling of `vets/vetList.html`.
- Confirmation/reminder emails are plain text (no HTML templates) for this PRD; an HTML template is a follow-up.

## 7. Technical Considerations

- **Spring Boot 3.x + Spring MVC + Thymeleaf + Spring Data JPA** are already in use; no new frameworks are introduced beyond `spring-boot-starter-mail`.
- **Database schema** must be updated in three places (`h2`, `mysql`, `postgres`). H2 is the dev/test default — verify the app boots cleanly with the updated `schema.sql` and `data.sql`.
- **Seed data** (`db/*/data.sql`): add a handful of `SCHEDULED` appointments for the next few weekdays so the staff console and owner page are non-empty in demos.
- **Mail in tests** must not hit the network. Use a `@TestConfiguration` providing a stub `JavaMailSender`, or rely on Spring Boot's `spring-boot-starter-mail` test slice with `spring.mail.protocol=smtps` disabled.
- **Race condition on slot booking (FR-5):** use a unique constraint on `(vet_id, start_time)` *filtered to status = SCHEDULED* where the DB supports partial indexes (Postgres). For MySQL/H2, rely on the application-level conflict check inside a transaction. Document the constraint difference per DB in code comments.
- **Build:** the project has both Maven (`pom.xml`) and Gradle (`build.gradle`) — every dependency addition must go into both, matching the existing pattern.
- **Clock injection** in `AppointmentNotificationService` and the slot service (`Clock` bean) so tests can drive deterministic times.

## 8. Success Metrics

- **Adoption:** ≥ 40% of new appointments created via the self-service form (vs. staff console) within 60 days post-launch.
- **No-show rate:** drop by 20% vs. baseline (attributed to 24h reminder).
- **Conflict errors:** < 1% of submitted booking forms hit the "slot just taken" error (race-condition signal).
- **Performance:** booking-form page load < 300ms p95 on H2/MySQL with 10k appointments seeded.

## 9. Open Questions

- **Fulfilling an appointment as a `Visit`.** When a vet sees the pet, who marks the appointment `COMPLETED` and how does that relate to creating a `Visit` record? Likely a follow-up PRD — but US-009's `status` enum already includes `COMPLETED` so the data model is ready.
- **Authentication.** Without auth, anyone can cancel anyone's appointment via URL guessing. Acceptable for this codebase's current model (which has the same property for owners/visits) but should be flagged for the auth-effort PRD.
- **Email provider.** Which SMTP host/account does the clinic use in production? Configuration only — does not block development on the H2 profile.
- **Server time zone.** Production deployments may span time zones. FR-13 punts on this; revisit if real customers ship.
- **Cancellation window.** Should owners be allowed to cancel < 2 hours before an appointment? PRD currently allows cancellation up to `startTime`; product to confirm.
