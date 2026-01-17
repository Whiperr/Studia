[podzial_pracy_zespolu.md](https://github.com/user-attachments/files/24691218/podzial_pracy_zespolu.md)
# Podział pracy zespołu - Aplikacja "Pracainz"

## Opis projektu

**Pracainz** to kompleksowy system HR do zarządzania pracownikami, urlopami, szkoleniami, sprzętem i umowami o pracę.

---

## 📋 Podział pracy na 3 osoby (wg modułów)

---

## 👤 Osoba 1: Moduł Urlopów i Umów o Pracę

### Zakres odpowiedzialności:

| Komponent | Pliki |
|-----------|-------|
| **Wnioski urlopowe** | `app/Http/Controllers/Leave/UserController.php`, `ManagerController.php`, `HrController.php`, `DeputyController.php` |
| **Modele urlopów** | `app/Models/LeaveRequest.php`, `LeaveRequestAction.php`, `UserLeaveInfo.php` |
| **Enumy statusów** | `app/Http/Controllers/Leave/Enums/RequestStatus.php`, `RequestType.php`, `RequestActionType.php` |
| **Umowy o pracę** | `app/Http/Controllers/Contract/EmploymentContractController.php`, `HrEmploymentContractController.php` |
| **Model umów** | `app/Models/EmploymentContract.php` |
| **Generowanie PDF** | `app/Services/Contracts/ContractPdfGenerator.php` |
| **Obliczanie dni** | `app/Helpers/Holidays.php` |
| **Widoki** | `resources/views/leave/` (20 plików), `resources/views/contracts/` (5 plików) |

### Przykładowe problemy:

1. **Wielopoziomowy workflow akceptacji** - wniosek musi przejść przez kierownika, akceptora i HR zanim zostanie zaakceptowany.
2. **Rezerwacja dni urlopowych** - przy składaniu wniosku dni muszą być zarezerwowane, a przy odrzuceniu zwolnione.
3. **Obliczanie dni roboczych** - konieczność pominięcia weekendów i polskich świąt.
4. **Walidacja nakładających się urlopów** - sprawdzenie czy pracownik nie ma już urlopu w wybranym terminie.

---

### Pytania i odpowiedzi:

#### 1. Opisz przepływ wniosku urlopowego od złożenia do zatwierdzenia.

**Odpowiedź:**

Przepływ wniosku urlopowego oparty jest na enumie statusów w pliku `app/Http/Controllers/Leave/Enums/RequestStatus.php`:

```php
enum RequestStatus: int {
    case WaitingForManagerAcceptance = 0;  // Oczekuje na kierownika
    case WaitingForHrAcceptance = 1;       // Oczekuje na HR
    case WaitingForAcceptorAcceptance = 2; // Oczekuje na akceptora
    case Accepted = 5;                      // Zaakceptowany
    case ManagerRejection = 10;            // Odrzucony przez kierownika
    case HrRejection = 11;                 // Odrzucony przez HR
    case Withdrawn = 12;                   // Wycofany
    case AcceptorRejection = 13;           // Odrzucony przez akceptora
}
```

**Workflow:**
1. **Pracownik składa wniosek** → `UserController::create()` (linia 24-165) ustawia status `WaitingForManagerAcceptance`
2. **Kierownik akceptuje** → `ManagerController::decision()` (linia 74-188) zmienia status na `WaitingForHrAcceptance`
3. **HR akceptuje** → `HrController::decision()` zmienia status na `Accepted`

Przy każdej zmianie tworzony jest wpis w `LeaveRequestAction` (linia 136-141 w `UserController.php`):
```php
$action = new LeaveRequestAction;
$action->request_id = $leaveRequest->id;
$action->type = RequestActionType::Created;
$action->user_id = $user->id;
$action->save();
```

---

#### 2. Jak obsługujecie rezerwację i zwalnianie dni urlopowych?

**Odpowiedź:**

System rezerwacji znajduje się w `UserController::create()` (linie 106-146) w transakcji bazodanowej:

```php
DB::transaction(function () use (...) {
    $leaveInfo = $user->leaveInfo()->lockForUpdate()->first();
    
    if ($requiresPool && $leaveInfo->available_days < $durationDays) {
        throw ValidationException::withMessages([
            'leave_days' => 'Brak wystarczającej liczby dni urlopowych.',
        ]);
    }
    
    // ... tworzenie wniosku ...
    
    if ($requiresPool) {
        $leaveInfo->reserveDays($durationDays);
    }
});
```

Przy wycofaniu wniosku (`UserController::abort()`, linie 167-206):
```php
if ($leaveRequest->requiresLeaveBalance()) {
    $leaveInfo = $leaveRequest->user->leaveInfo()->lockForUpdate()->first();
    if ($leaveInfo) {
        $leaveInfo->releaseReservedDays($duration);
    }
}
```

Metoda `requiresLeaveBalance()` w `LeaveRequest.php` (linia 54-58) sprawdza typ urlopu:
```php
public function requiresLeaveBalance(): bool
{
    return $this->type === RequestType::Leisure
        || $this->type === RequestType::OnDemand;
}
```

---

#### 3. Jak obliczacie dni robocze z pominięciem świąt?

**Odpowiedź:**

Klasa `app/Helpers/Holidays.php` używa biblioteki **Yasumi** do obsługi polskich świąt:

```php
public static function countWorkingDays($startDate, $endDate): int
{
    $start = Carbon::parse($startDate)->startOfDay();
    $end = Carbon::parse($endDate)->startOfDay();
    
    $holidayDates = static::holidayDatesBetween($start, $end);
    $workdays = 0;

    foreach (CarbonPeriod::create($start, $end) as $date) {
        if ($date->isWeekend()) {
            continue;
        }
        if ($holidayDates->contains($date->toDateString())) {
            continue;
        }
        $workdays++;
    }
    return $workdays;
}
```

Święta pobierane są z Yasumi + dodatkowe dni z tabeli `additional_holidays` (linia 59-75):
```php
protected static function holidayDatesBetween(Carbon $start, Carbon $end): Collection
{
    foreach ($years as $year) {
        $yearHolidays = collect(Yasumi::create('Poland', $year)->getHolidayDates());
        $dates = $dates->merge($yearHolidays);
    }
    
    $additional = AdditionalHoliday::pluck('special_date');
    return $dates->merge($additional)->unique()->values();
}
```

---

#### 4. Jak działa podpisywanie umów o pracę?

**Odpowiedź:**

W `EmploymentContractController::sign()` (linie 66-80):

```php
public function sign(EmploymentContract $contract)
{
    $this->authorize('sign', $contract);

    if ($contract->isSigned()) {
        abort(403, 'Umowa została już podpisana.');
    }

    $contract->update([
        'signed_at' => now(),
        'signed_by_id' => Auth::id(),
    ]);

    return redirect()->back()->with('success', 'Umowa została podpisana.');
}
```

Autoryzacja przez Policy (`app/Policies/EmploymentContractPolicy.php`) sprawdza czy użytkownik może podpisać umowę.

---

#### 5. Jakie typy urlopów obsługuje system?

**Odpowiedź:**

W pliku `app/Http/Controllers/Leave/Enums/RequestType.php` zdefiniowane są typy:

- `Leisure` - urlop wypoczynkowy (wymaga puli dni)
- `OnDemand` - urlop na żądanie (wymaga puli dni)
- `Circumstantial` - urlop okolicznościowy (wymaga podania powodu)
- `Paternity` - urlop ojcowski
- `Other` - inny
- `B2B` - dla kontraktów B2B (ukryty dla pracowników)

Filtrowanie typów w `UserController::allowedLeaveTypes()` (linia 274-279):
```php
private function allowedLeaveTypes()
{
    return collect(RequestType::cases())
        ->reject(fn ($type) => $type === RequestType::B2B)
        ->values();
}
```

---

## 👤 Osoba 2: Moduł Stanowisk i Stawek Historycznych

### Zakres odpowiedzialności:

| Komponent | Pliki |
|-----------|-------|
| **Stanowiska** | `app/Http/Controllers/Position/PositionController.php`, `UserPositionController.php`, `PositionHistoryController.php` |
| **Modele stanowisk** | `app/Models/Position.php`, `UserPosition.php` |
| **Stawki historyczne** | `app/Http/Controllers/Rate/HistoricalRateController.php` |
| **Model stawek** | `app/Models/HistoricalRate.php` |
| **Panel HR** | `app/Http/Controllers/Hr/HrController.php` |
| **Eksport raportów** | `app/Http/Controllers/ReportController.php`, `app/Exports/` |
| **Widoki** | `resources/views/position/` (9 plików), `resources/views/Rates/` (5 plików), `resources/views/hr/` (6 plików) |

### Przykładowe problemy:

1. **Ścieżki kariery** - stanowiska mogą tworzyć łańcuch awansów, trzeba było zabezpieczyć przed cyklami.
2. **Historia stanowisk** - śledzenie wszystkich zmian stanowisk pracownika z datami.
3. **Stawki historyczne** - przechowywanie historii zmian stawek z datą obowiązywania.
4. **Eksport do Excel** - generowanie raportów z danymi kadrowymi.

---

### Pytania i odpowiedzi:

#### 1. Jak działa ścieżka kariery (career path) i jak zabezpieczyliście przed cyklami?

**Odpowiedź:**

Model `Position.php` ma relację `next_position_id` wskazującą na następne stanowisko w ścieżce awansu.

W `PositionController::update()` (linie 105-131) przed zapisem sprawdzany jest cykl:

```php
if (isset($validated['next_position_id']) && $validated['next_position_id']) {
    if ($this->wouldCreateCycle($position->id, $validated['next_position_id'])) {
        return back()->withErrors([
            'next_position_id' => 'Nie można ustawić tego stanowiska jako następne - utworzyłoby to cykl w ścieżce rozwoju.'
        ]);
    }
}
```

Algorytm wykrywania cykli (`PositionController::wouldCreateCycle()`, linie 157-175):

```php
private function wouldCreateCycle($positionId, $nextPositionId, $visited = [])
{
    if ($positionId == $nextPositionId) {
        return true;
    }

    if (in_array($nextPositionId, $visited)) {
        return true;
    }

    $visited[] = $nextPositionId;
    $nextPosition = Position::find($nextPositionId);

    if ($nextPosition && $nextPosition->next_position_id) {
        return $this->wouldCreateCycle($positionId, $nextPosition->next_position_id, $visited);
    }

    return false;
}
```

To rekurencyjne przeszukiwanie grafu - sprawdza czy dodanie połączenia nie utworzy pętli.

---

#### 2. Jak wygląda przypisywanie stanowiska do użytkownika?

**Odpowiedź:**

W `UserPositionController::store()` tworzy się wpis `UserPosition` z polami:
- `user_id` - pracownik
- `position_id` - stanowisko
- `start_date` - data rozpoczęcia
- `end_date` - data zakończenia (null = aktualne)
- `is_current` - czy jest to aktualne stanowisko

Model `UserPosition.php` zawiera logikę:
```php
public function scopeCurrent($query)
{
    return $query->where('is_current', true);
}

public function end(): void
{
    $this->update([
        'end_date' => now(),
        'is_current' => false,
    ]);
}
```

Przy przypisaniu nowego stanowiska, poprzednie jest automatycznie kończone.

---

#### 3. Jak działa historia stawek pracownika?

**Odpowiedź:**

Model `HistoricalRate.php` przechowuje:
- `user_id` - pracownik
- `rate` - stawka
- `effective_from` - data obowiązywania

W `User.php` relacja (linia 173-176):
```php
public function historicalRates()
{
    return $this->hasMany(HistoricalRate::class)->orderBy('effective_from', 'desc');
}
```

`HistoricalRateController::store()` tworzy nowy wpis z datą obowiązywania - nie nadpisuje poprzednich, dzięki czemu zachowana jest pełna historia.

---

#### 4. Jak ograniczyliście dostęp do funkcji HR?

**Odpowiedź:**

W `PositionController::ensureHrAccess()` (linie 177-184):

```php
private function ensureHrAccess(): ?RedirectResponse
{
    if (! Auth::user()?->is_hr) {
        return redirect()->route('position.index')
            ->with('error', 'Nie masz uprawnień do zarządzania stanowiskami.');
    }
    return null;
}
```

Wywoływane na początku metod `create()`, `store()`, `edit()`, `update()`, `destroy()`.

Dodatkowo w `routes/web.php` middleware:
```php
Route::prefix('/position')->name('position.')->middleware('role:DepartmentCreatorOrHr')->group(function () {
    Route::get('/create', [PositionController::class, 'create'])->middleware('role:hr')->name('create');
    // ...
});
```

---

#### 5. Jak zapobiegacie usunięciu stanowiska z przypisanymi użytkownikami?

**Odpowiedź:**

W `PositionController::destroy()` (linie 136-152):

```php
public function destroy(Position $position)
{
    if ($redirect = $this->ensureHrAccess()) {
        return $redirect;
    }

    if ($position->userPositions()->exists()) {
        return back()->withErrors([
            'delete' => 'Nie można usunąć stanowiska, które ma przypisanych użytkowników. Najpierw usuń przypisania.'
        ]);
    }

    $position->delete();
    return redirect()->route('position.index')->with('success', 'Stanowisko zostało usunięte!');
}
```

---

## 👤 Osoba 3: Moduł Szkoleń i Sprzętu

### Zakres odpowiedzialności:

| Komponent | Pliki |
|-----------|-------|
| **Szkolenia** | `app/Http/Controllers/Training/TrainingManagementController.php`, `TrainingController.php`, `TrainingParticipantController.php`, `TrainingLocationController.php` |
| **Modele szkoleń** | `app/Models/Training.php`, `TrainingLocation.php`, `TrainingParticipant.php` |
| **Sprzęt** | `app/Http/Controllers/Equipment/EquipmentItemController.php`, `EquipmentCategoryController.php`, `EquipmentAssignmentController.php`, `EquipmentDashboardController.php` |
| **Modele sprzętu** | `app/Models/EquipmentItem.php`, `EquipmentCategory.php`, `EquipmentAssignment.php` |
| **Panel admina** | `app/Http/Controllers/Admin/AdminController.php` |
| **Widoki** | `resources/views/trainings/` (7 plików), `resources/views/equipment/` (10 plików), `resources/views/admin/` (7 plików) |

### Przykładowe problemy:

1. **Synchronizacja uczestników** - przy edycji szkolenia trzeba dodać nowych i usunąć tych, którzy zostali wypisani.
2. **Uprawnienia kierowników** - kierownik widzi tylko szkolenia swoich podwładnych.
3. **Przypisywanie sprzętu** - jeden element może być przypisany tylko do jednego pracownika.
4. **Akceptacja sprzętu** - pracownik musi potwierdzić otrzymanie sprzętu.

---

### Pytania i odpowiedzi:

#### 1. Jak działa system szkoleń - od utworzenia do zarządzania uczestnikami?

**Odpowiedź:**

**Tworzenie szkolenia** - `TrainingManagementController::store()` (linie 67-96):
```php
$training = Training::create(array_merge(
    Arr::only($validated, [
        'title', 'description', 'agenda', 'start_at', 'end_at',
        'location_id', 'location_details', 'trainer_name', 'trainer_email', 'allow_self_enroll',
    ]),
    ['created_by' => $user->id]
));

$this->syncDepartments($training, $validated['department_ids'] ?? []);
$this->syncParticipants($training, $validated['participant_ids'] ?? [], $user);
```

**Synchronizacja uczestników** - `TrainingManagementController::syncParticipants()` (linie 232-263):
```php
private function syncParticipants(Training $training, array $participantIds, User $assigner): void
{
    $currentAssigned = $training->participants()->whereNotNull('assigned_by')->get();
    $currentUserIds = $currentAssigned->pluck('user_id');
    
    $toAdd = $participantIds->diff($currentUserIds);
    $toRemove = $currentAssigned->whereNotIn('user_id', $participantIds)->pluck('id');

    foreach ($toAdd as $userId) {
        TrainingParticipant::updateOrCreate(
            ['training_id' => $training->id, 'user_id' => $userId],
            ['assigned_by' => $assigner->id, 'status' => TrainingParticipant::STATUS_ASSIGNED]
        );
    }

    if ($toRemove->isNotEmpty()) {
        $training->participants()->whereIn('id', $toRemove)->delete();
    }
}
```

---

#### 2. Jak różnią się uprawnienia HR i kierownika w module szkoleń?

**Odpowiedź:**

W `TrainingManagementController::index()` (linie 19-45):

```php
if ($user->is_hr) {
    // HR sees everything
} elseif ($this->hasDirectReports($user)) {
    $subordinateIds = $user->subordinates()->pluck('id');
    $query->where(function ($builder) use ($user, $subordinateIds) {
        $builder->where('created_by', $user->id)
            ->orWhereHas('participants', function ($participantQuery) use ($subordinateIds) {
                $participantQuery->whereIn('user_id', $subordinateIds);
            });
    });
} else {
    abort(403, 'Nie masz uprawnień do przeglądania tej listy.');
}
```

**HR** widzi wszystkie szkolenia. **Kierownik** widzi tylko:
- szkolenia które sam utworzył
- szkolenia w których uczestniczą jego podwładni

Przy tworzeniu szkolenia (`validatePayload()`, linie 202-217):
```php
if (! $user->is_hr) {
    $allowedUserIds = $user->subordinates()->pluck('id');
    // Kierownik może przypisać tylko swoich podwładnych
    
    if (! empty($validated['department_ids'])) {
        throw ValidationException::withMessages([
            'department_ids' => 'Przełożony nie może ograniczać szkoleń do departamentów.',
        ]);
    }
}
```

---

#### 3. Jak działa przypisywanie sprzętu do pracownika?

**Odpowiedź:**

Model `EquipmentAssignment.php` przechowuje:
- `equipment_item_id` - element sprzętu
- `user_id` - pracownik
- `assigned_by` - kto przypisał
- `assigned_at` - kiedy przypisano
- `accepted_at` - kiedy zaakceptowano (null = nie zaakceptowano)
- `returned_at` - kiedy zwrócono (null = aktywne przypisanie)

**Scope aktywnych przypisań** (linia 53-56):
```php
public function scopeActive($query)
{
    return $query->whereNull('returned_at');
}
```

**Sprawdzenie akceptacji** (linia 58-61):
```php
public function isAccepted(): bool
{
    return ! is_null($this->accepted_at);
}
```

---

#### 4. Jak zabezpieczyliście przed usunięciem sprzętu który jest przypisany?

**Odpowiedź:**

W `EquipmentItemController::destroy()` (linie 61-74):

```php
public function destroy(EquipmentItem $item): RedirectResponse
{
    if ($item->activeAssignment()->exists()) {
        return redirect()
            ->route('hr.equipment.items.index')
            ->withErrors(['item' => 'Nie można usunąć zasobu przypisanego do pracownika.']);
    }

    $item->delete();
    return redirect()->route('hr.equipment.items.index')->with('success', 'Zasób został usunięty.');
}
```

Relacja `activeAssignment()` w `EquipmentItem.php` używa scope'a `active()` który filtruje po `returned_at IS NULL`.

---

#### 5. Jak pracownik akceptuje otrzymany sprzęt?

**Odpowiedź:**

W `MyEquipmentController::accept()`:

```php
public function accept(EquipmentAssignment $assignment)
{
    if ($assignment->user_id !== Auth::id()) {
        abort(403);
    }
    
    if ($assignment->isAccepted()) {
        return redirect()->back()->with('info', 'Sprzęt został już zaakceptowany.');
    }
    
    $assignment->update([
        'accepted_at' => now(),
        'accepted_by' => Auth::id(),
    ]);
    
    return redirect()->back()->with('success', 'Sprzęt został zaakceptowany.');
}
```

Widok `/equipment/my` pokazuje wszystkie przypisania użytkownika z przyciskiem "Akceptuj" dla tych bez `accepted_at`.

---

## 🎓 Pytania dla całego zespołu

#### 1. Jak działają role i uprawnienia w systemie?

**Odpowiedź:**

System ról oparty jest na:
- Middleware `EnsureUserHasRole.php` - sprawdza rolę przed dostępem do trasy
- Model `Role.php` i `UserRole.php` - relacja wiele-do-wielu
- Pole `is_hr` w `User.php` - szybkie sprawdzenie uprawnień HR

W `routes/web.php`:
```php
Route::prefix('/admin')->middleware('role:DepartmentCreator')->name('admin.')->group(function () {...});
Route::prefix('/leave/hr')->middleware('role:hr')->name('leave.hr.')->group(function () {...});
Route::prefix('/leave/manager')->middleware('role:manager')->name('leave.manager.')->group(function () {...});
```

---

#### 2. Jak obsługujecie relacje przełożony-podwładny?

**Odpowiedź:**

W `User.php` (linie 78-99):
```php
public function subordinates()
{
    return $this->hasMany(User::class, 'manager_id', 'id')
        ->where('name', '!=', static::SYSTEM_USERNAME)
        ->withActiveContract();
}

public function mainManager()
{
    return $this->belongsTo(User::class, 'manager_id', 'id');
}

public function mainDeputy()
{
    return $this->belongsTo(User::class, 'deputy_id', 'id');
}
```

Pola `manager_id` i `deputy_id` w tabeli `users` wskazują na przełożonego i zastępcę.

---

#### 3. Jak filtrowaliście użytkowników nieaktywnych?

**Odpowiedź:**

Scope `visible()` w `User.php` (linie 59-69):
```php
public function scopeVisible($query)
{
    return $query
        ->where('name', '!=', static::SYSTEM_USERNAME)
        ->where(function ($builder) {
            $builder->whereDoesntHave('employmentContracts')
                ->orWhereHas('employmentContracts', function ($contractQuery) {
                    $contractQuery->active();
                });
        });
}
```

Ukrywa użytkownika systemowego `admin` i pokazuje tylko tych z aktywną umową lub bez żadnej umowy.

---

## 📊 Statystyki projektu

| Metryka | Wartość |
|---------|---------|
| Modele Eloquent | 21 |
| Kontrolery | ~30 |
| Migracje | 56 |
| Widoki Blade | ~80 |
| Enumy | 3 (RequestStatus, RequestType, RequestActionType) |
