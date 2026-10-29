# פרומפטים ל-Base44

מדביקים כל פרומפט בצ'אט של Base44 לפי הסדר. אחרי כל אחד עושים את ה"בדיקה" שכתובה ב-[tasks.md](tasks.md), ורק אז ממשיכים. אם משהו יוצא לא נכון, מתקנים בפרומפט קצר וממוקד ("In the Animal entity, change X to Y"), ולא מתחילים מחדש.

---

## P1: אפליקציה וישויות

```
Build a web app called "חברים על ארבע" (Friends on Four) for an animal rescue nonprofit that runs a shelter for dogs and cats plus a network of foster families.

The entire UI must be in Hebrew, right-to-left (dir="rtl"). Warm, clean design: white background, warm orange primary color, rounded cards, friendly animal imagery.

In this step create ONLY the data model, no pages yet. Entities:

1. Extend the built-in User entity with:
   - app_role: enum [admin, vet, coordinator, volunteer, foster, adopter], default "adopter"
   - phone: text
   - address: text

2. Animal:
   - name (text, required), species (enum: dog, cat), breed (text), sex (enum: male, female)
   - birth_date_estimate (date), size (enum: small, medium, large), energy_level (enum: low, medium, high)
   - photos (array of image URLs), public_description (long text), intake_date (date)
   - location_type (enum: shelter, foster, adopted), foster_id (reference to User, only when location_type = foster)
   - status (enum: intake, in_treatment, available, pending_adoption, adopted)
   - is_blocked (boolean, default false; system-managed, not editable in forms)
   - good_with_kids, good_with_dogs, good_with_cats, house_trained (each enum: yes, no, unknown, default unknown)

3. MedicalRecord:
   - animal_id (reference to Animal, required)
   - type (enum: vaccination, deworming, flea_treatment, surgery, medication, checkup, special_diet)
   - description (text), medication (text), start_date (date), end_date (date)
   - no_leave (boolean: animal must not leave / treatment must not be interrupted)
   - next_due_date (date), performed_by (reference to User)
   - status (enum: scheduled, active, done)

4. BehaviorReport:
   - animal_id (reference to Animal), reporter_id (reference to User), report_date (date)
   - house_trained, good_with_kids, good_with_dogs, good_with_cats, leash_walking (each enum: yes, no, unknown)
   - energy_observed (enum: low, medium, high), notes (long text), photo (image)

5. AdoptionApplication:
   - applicant_id (reference to User), animal_id (reference to Animal, optional)
   - home_type (enum: apartment, apartment_with_balcony, house_with_yard)
   - has_kids, has_dogs, has_cats (boolean), hours_alone_per_day (number)
   - experience (enum: none, some, experienced)
   - preferred_size (enum: small, medium, large, any), preferred_energy (enum: low, medium, high, any)
   - motivation (long text)
   - status (enum: new, in_review, meeting_scheduled, approved, adopted, rejected; default new)
   - assigned_coordinator (reference to User)

6. MatchRecommendation:
   - application_id (reference to AdoptionApplication), animal_id (reference to Animal)
   - score (number 0-100), reasoning (long text), warnings (text)

Show all enum values to users with Hebrew labels (e.g. dog = כלב, available = זמין לאימוץ).
```

## P2: נתוני דמו

```
Seed the database with realistic Hebrew demo data:

- 15 animals (10 dogs, 5 cats) with Hebrew names, varied sizes, energy levels and statuses:
  9 available, 2 in_treatment, 2 pending_adoption, 1 intake, 1 adopted.
  Use real animal photos from Unsplash. 5 of them are with foster families (location_type = foster).
  Include "לוקי" – a large, high-energy young dog, available, in foster.
  Include "בל" – a medium calm female dog, available, in the shelter.
- 5 users with app_role = foster; link each foster animal to one of them.
- 25 MedicalRecords: vaccinations, deworming etc. Include:
  - one active antibiotic course with no_leave = true on one in_treatment dog (end_date 5 days from today)
  - 3 vaccinations whose next_due_date is within the next 7 days
  - 2 vaccinations whose next_due_date already passed (overdue)
- 10 BehaviorReports from the foster families, with Hebrew notes describing behavior.
- 8 AdoptionApplications with varied statuses (4 new). Include one applicant living in a small apartment, alone 9 hours a day, no experience – a deliberately poor fit for לוקי.
```

## P3: הרשאות

```
Implement role-based access using User.app_role. Enforce it at the data level (entity security rules / row level security), not only by hiding UI.

- Guests (not logged in): can see only the public pages. From Animal they may see only animals with status = available and is_blocked = false, and only these fields: name, species, breed, sex, birth_date_estimate, size, energy_level, photos, public_description, good_with_kids, good_with_dogs, good_with_cats, house_trained. No medical data, no foster_id. If field-level restriction is not possible, create a backend function getPublicAnimals that returns only those fields and use it for public pages.
- adopter: same as guest, plus create AdoptionApplication and read/update ONLY applications where applicant_id = current user (and may not change status).
- foster: read Animal and MedicalRecord only where Animal.foster_id = current user; create BehaviorReport for those animals and read their own reports.
- volunteer: read all Animals, MedicalRecords and BehaviorReports; create BehaviorReport.
- vet: full access to MedicalRecord; read and update Animal; read BehaviorReport.
- coordinator: read and update Animal; full access to AdoptionApplication and MatchRecommendation; read MedicalRecord and BehaviorReport.
- admin: full access to everything, including managing users and their app_role.

Navigation menu must show only the pages the current role may use. If a user opens a URL they are not allowed to see, show a Hebrew "אין לך הרשאה לצפות בדף זה" message.
```

## P4: מסכים ציבוריים

```
Build the public pages (Hebrew, RTL, mobile-friendly):

1. Landing page: hero with a warm headline and photo, two CTAs: "מצאו חבר לחיים" (to the gallery) and "הצטרפו כמשפחת אומנה". Sections: how adoption works (3 steps), 3 featured available animals, success stories (3 testimonials), FAQ (5 questions about adoption), footer with contact info.
2. Adoption gallery: cards of available, non-blocked animals with photo, name, age (computed from birth_date_estimate), size. Filters: species, size, age range, energy level, good with kids / dogs / cats.
3. Public animal profile: photo carousel, description, behavior traits as friendly badges (e.g. "מסתדר עם ילדים ✓"), and a button "אני רוצה לאמץ" that requires login and opens the application form with this animal preselected.
4. Adoption application form (logged-in): all AdoptionApplication fields with Hebrew labels, validation, success message.
5. "הבקשות שלי": the adopter's applications with a status timeline (התקבלה ← בבדיקה ← נקבעה פגישה ← אושרה / אומץ / נדחתה).
```

## P5: מסכים פנימיים

```
Build the internal staff pages:

1. Dashboard (admin, coordinator): KPI cards – animals by status, animals by location (shelter / foster), open applications, overdue medical tasks, blocked animals. Below: a table of all animals with search and filters (status, location, species, blocked).
2. Internal animal profile (staff, and foster for their own animals): tabs "פרטים", "רפואי" (timeline of MedicalRecords, newest first, with add button for vet), "התנהגות" (BehaviorReports), "בקשות" (applications for this animal, coordinator only). When is_blocked = true, show a prominent red banner: "בטיפול פעיל – אסור לצאת מהמחסה / למסור לאימוץ".
3. Medical tasks (vet): list of MedicalRecords whose next_due_date is within 7 days or already passed and status != done, sorted by date, overdue highlighted in red, with a "סמן כבוצע" action.
4. Applications board (coordinator): Kanban by status, drag to change status, card shows applicant name, requested animal and best match score if exists.
5. Foster area (foster): "בעלי החיים שלי" cards, upcoming medical due dates for them, and a weekly behavior report form.
```

## P6: לוגיקת חסימה

```
Add a backend rule: whenever a MedicalRecord is created or updated, recompute the related Animal:
- is_blocked = true if the animal has ANY MedicalRecord with no_leave = true AND status = active AND (end_date is empty OR end_date >= today); otherwise false.
- If is_blocked becomes true and the animal status is available or pending_adoption, set status = in_treatment.
Also block in the UI and in the backend any attempt to set status = available or pending_adoption while is_blocked = true, with the Hebrew error "לא ניתן – בעל החיים בטיפול פעיל שאסור להפסיק".
```

## P7: סוכן התאמה

```
Create an AI agent named "סוכן התאמת אימוץ".

Purpose: given one AdoptionApplication, evaluate fit against available animals and write recommendations.

Tools / permissions (least privilege):
- READ: Animal (only status = available and is_blocked = false), BehaviorReport, AdoptionApplication.
- CREATE: MatchRecommendation only.
- No access to MedicalRecord or User contact details. Must NOT update application status or any animal.

Instructions:
1. Read the application and, if animal_id is set, that animal plus its BehaviorReports.
2. Score fit 0-100 using: size and energy vs home_type and hours_alone_per_day; good_with_kids / dogs / cats vs has_kids / has_dogs / has_cats; adopter experience vs energy level; preferred_size / preferred_energy. Prefer behavior observed in BehaviorReports over the static fields.
3. Create a MatchRecommendation for the requested animal (if any) and for the 3 best other available animals. Only recommend animal ids that exist in the data you read – never invent animals.
4. reasoning: 2-3 short Hebrew sentences. warnings: concrete risks in Hebrew (e.g. "כלב אנרגטי ודירה קטנה ללא חצר").
5. The final decision is always human – say so at the end of your answer.

In the application details page (coordinator), add a button "הפעל סוכן התאמה" that runs the agent on that application and then shows the resulting recommendations as cards sorted by score, with links to the animals.
```

## P8: אוטומציות ומיילים

```
Add scheduled automations (daily at 08:00) that send emails using the built-in email integration, in Hebrew:
1. Medical reminders: for each MedicalRecord with status != done and next_due_date within the next 7 days or overdue – email the foster family (if the animal is in foster) and all users with app_role = vet. Include animal name, treatment type and due date.
2. Foster weekly report: for each foster user who has an animal in foster and no BehaviorReport for it in the last 7 days – email a friendly reminder with a link to the foster area.
3. When an AdoptionApplication status changes – email the applicant with the new status in Hebrew.
Add a "שלח תזכורות עכשיו" button on the dashboard (admin only) to trigger reminders manually for the demo.
```

---

# פרומפטים מאוחדים אחרי סבב בדיקה 1 (2026-10-29)

הבדיקה של P3 עד P5 מצאה פערים (ראו [verification.md](../../docs/verification.md)). כדי לחסוך קרדיטים, התיקונים אוחדו עם השלבים הבאים. **P6+ מחליף את P6, ו-P7+ מחליף את P7.** הסדר: P6+ ← P7+ ← P9 ← P8 (בונוס).

לפני P6+ צריך לתקן ידנית ב-Data (בלי AI): את 5 החיסונים הקרובים או באיחור ל-status = scheduled, ואת התמונות של לוקי וסנדי (T1.4, T1.5).

## P6+: חסימה, כרטיס פנימי ומשימות רפואיות

```
Fixes and next step. Keep everything that already works.

1. Blocking logic (backend): whenever a MedicalRecord is created or updated, recompute the related Animal:
   - is_blocked = true if the animal has ANY MedicalRecord with no_leave = true AND status = active AND (end_date is empty OR end_date >= today); otherwise false.
   - If is_blocked becomes true and status is available or pending_adoption, set status = in_treatment.
   - Reject (UI and backend) any attempt to set status = available or pending_adoption while is_blocked = true, with the Hebrew error "לא ניתן – בעל החיים בטיפול פעיל שאסור להפסיק".

2. Internal animal profile page at /dashboard/animals/:id (replace the current edit dialog; keep an "ערוך" button inside the page). Tabs:
   - "פרטים": all fields, editable by admin/coordinator/vet. Include a foster selector (users with app_role = foster) that sets foster_id and foster_name. When foster_id changes, update foster_id on all MedicalRecords of this animal too (RLS depends on it).
   - "רפואי": MedicalRecords timeline, newest first, with "הוסף טיפול" for vet/admin.
   - "התנהגות": BehaviorReports, newest first.
   - "בקשות": AdoptionApplications for this animal (coordinator/admin only).
   When is_blocked = true show a prominent red banner at the top: "בטיפול פעיל – אסור לצאת מהמחסה / למסור לאימוץ". Clicking a row in the dashboard animals table opens this page.

3. Medical tasks page (vet): show ONLY MedicalRecords with status != done AND next_due_date <= today + 7 days (including overdue). Sort by next_due_date ascending. Overdue rows highlighted in red with "באיחור X ימים". "סמן כבוצע" sets status = done and removes the row.
```

## P7+: סוכן התאמה ולוח Kanban

```
1. Create an AI agent named "סוכן התאמת אימוץ".
   Tools / permissions (least privilege): READ Animal (only status = available and is_blocked = false), BehaviorReport, AdoptionApplication. CREATE MatchRecommendation only. No access to MedicalRecord or User contact details. Must NOT update any application or animal.
   Instructions:
   a. Read the application and, if animal_id is set, that animal plus its BehaviorReports.
   b. Score fit 0-100: size and energy vs home_type and hours_alone_per_day; good_with_kids/dogs/cats vs has_kids/dogs/cats; adopter experience vs energy; preferred_size / preferred_energy. Prefer behavior observed in BehaviorReports over static fields.
   c. Create a MatchRecommendation for the requested animal (if any) and for the 3 best other available animals. Only use animal ids from the data you read – never invent animals.
   d. reasoning: 2-3 short Hebrew sentences. warnings: concrete risks in Hebrew (e.g. "כלב אנרגטי ודירה קטנה ללא חצר").
   e. End by stating that the final decision is human.

2. Applications board (coordinator/admin): replace the list with a Kanban, one column per status (חדשה, בבדיקה, נקבעה פגישה, אושרה, אומץ, נדחתה), drag a card to change status. Card: applicant name, requested animal, best match score if exists.
   Clicking a card opens the application details with a button "הפעל סוכן התאמה" that runs the agent and then shows the recommendations as cards sorted by score, with links to the animals.
```

## P9: דשבורד, כרטיס ציבורי והרשאות

```
1. Dashboard: add KPI cards for animals by location (shelter / foster), overdue medical tasks (status != done and next_due_date < today), and blocked animals. Add table filters for location_type, species and is_blocked.

2. Public animal profile: add a section "מה משפחת האומנה מספרת" that summarizes the latest BehaviorReport (leash walking, energy, good with kids/dogs/cats, and the notes text, without reporter name). Return it through the public backend function (getPublicAnimals or a new getPublicAnimal) so guests never read BehaviorReport directly.

3. Permissions fixes (enforce in backend, not only UI):
   - BehaviorReport create: a foster may create a report only if Animal.foster_id = current user. Volunteer, vet and admin may create for any animal.
   - AdoptionApplication: an adopter may create and read their own applications, but may NOT change status or assigned_coordinator. Only coordinator/admin may change status. If the platform rules cannot restrict single fields, route adopter updates through a backend function that ignores those fields.
```
