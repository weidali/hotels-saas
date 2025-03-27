### **Архитектура мультитенантной системы для отелей (Laravel)**

#### **1. Основные принципы**
1. **Изоляция данных**: Каждый отель видит только своих клиентов и бронирования
2. **Общие ресурсы**: Единая кодовая база для всех отелей
3. **Гибкое управление**: Возможность добавлять новые отели без изменений кода

---

### **2. Реализация в Laravel**

#### **A. Модель данных**
```bash
php artisan make:model Hotel -m
php artisan make:model Client -m
```

**Миграции:**
```php
// hotels table
Schema::create('hotels', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('subdomain')->unique();
    $table->timestamps();
});

// clients table
Schema::create('clients', function (Blueprint $table) {
    $table->id();
    $table->foreignId('hotel_id')->constrained();
    $table->string('name');
    $table->string('phone');
    $table->timestamps();
});

// Добавляем hotel_id в bookings
Schema::table('bookings', function (Blueprint $table) {
    $table->foreignId('hotel_id')->constrained();
});
```

#### **B. Определение текущего отеля**
**Middleware TenantDetection:**
```bash
php artisan make:middleware TenantDetection
```

```php
public function handle($request, Closure $next)
{
    $host = $request->getHost();
    $subdomain = explode('.', $host)[0];
    
    $hotel = Hotel::where('subdomain', $subdomain)->firstOrFail();
    
    // Сохраняем отель в контейнере сервиса
    app()->instance('currentHotel', $hotel);
    
    return $next($request);
}
```

#### **C. Глобальные scope для моделей**
```php
// app/Models/Booking.php
protected static function booted()
{
    static::addGlobalScope('hotel', function (Builder $builder) {
        if ($hotel = app('currentHotel')) {
            $builder->where('hotel_id', $hotel->id);
        }
    });
}
```

#### **D. Настройка маршрутизации**
**routes/web.php:**
```php
Route::domain('{subdomain}.'.env('APP_DOMAIN'))
     ->middleware(['tenant'])
     ->group(function () {
         Route::get('/', [BookingController::class, 'index']);
     });
```

---

### **3. Авторизация пользователей**

#### **A. Роли в рамках отеля**
```bash
php artisan make:model HotelUser -m
```

**Структура таблицы:**
```php
Schema::create('hotel_users', function (Blueprint $table) {
    $table->id();
    $table->foreignId('hotel_id')->constrained();
    $table->foreignId('user_id')->constrained();
    $table->string('role'); // admin, manager, reception
    $table->timestamps();
});
```

#### **B. Политики доступа**
```bash
php artisan make:policy BookingPolicy --model=Booking
```

```php
public function viewAny(User $user)
{
    return $user->hotels()
        ->where('hotel_id', app('currentHotel')->id)
        ->exists();
}
```

---

### **4. Работа с поддоменами**

#### **Настройка DNS**
```
*.yourdomain.com A 127.0.0.1
```

#### **Конфигурация Laravel**
**.env:**
```
APP_URL=yourdomain.com
SESSION_DOMAIN=.yourdomain.com
```

---

### **5. Примеры запросов**

**Получение бронирований текущего отеля:**
```php
// Автоматически применяется scope
$bookings = Booking::all(); 

// Явное указание
$bookings = Booking::where('hotel_id', app('currentHotel')->id)->get();
```

**Создание нового клиента:**
```php
Client::create([
    'hotel_id' => app('currentHotel')->id,
    'name' => $request->name,
    'phone' => $request->phone
]);
```

---

### **6. Дополнительные механизмы**

#### **A. Кэширование данных отеля**
```php
Cache::remember("hotel_{$subdomain}", 3600, function () use ($subdomain) {
    return Hotel::with('settings')->where('subdomain', $subdomain)->first();
});
```

#### **B. Изоляция файлов**
```php
Storage::disk('uploads')
    ->put("hotels/{$hotel->id}/".$filename, $file);
```

---

### **7. Тестирование мультитенантности**

**.env.testing:**
```
APP_URL=localhost
SESSION_DOMAIN=null
```

**Тестовый случай:**
```php
public function test_hotel_isolation()
{
    $hotel1 = Hotel::factory()->create(['subdomain' => 'hotel1']);
    $hotel2 = Hotel::factory()->create(['subdomain' => 'hotel2']);
    
    $this->get('http://hotel1.app.test/bookings')
         ->assertSee($hotel1->bookings[0]->id)
         ->assertDontSee($hotel2->bookings[0]->id);
}
```

---

### **📌 Итоговая архитектура**
```
1. Уровень данных:
   - Глобальные scope для всех моделей
   - Связи через hotel_id

2. Уровень приложения:
   - Middleware для определения отеля
   - Политики для проверки доступа

3. Уровень инфраструктуры:
   - Поддоменная маршрутизация
   - Изолированное хранилище файлов
```

**Преимущества:**
- Полная изоляция данных между отелями
- Общие настройки для всех экземпляров
- Легкое добавление новых отелей

Для внедрения:
1. Настройте DNS и окружение
2. Примените миграции
3. Протестируйте с разными поддоменами
