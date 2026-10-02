# Part 08: OOP - Inheritance & Polymorphism
## ขั้นตอนที่ 71-80: การสืบทอดคลาสและ Polymorphism

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Inheritance และ class hierarchy
- ใช้ virtual, override, new keywords
- เข้าใจ Polymorphism
- ใช้ abstract classes
- รู้จัก sealed classes
- ใช้ base keyword

---

## ขั้นตอนที่ 71: Inheritance พื้นฐาน

### การสืบทอด Class
```csharp
// Base class (Parent/Super class)
class Animal
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string Sound { get; protected set; } = "";
    
    public Animal(string name, int age)
    {
        Name = name;
        Age = age;
    }
    
    // Virtual method - สามารถ override ได้
    public virtual void MakeSound()
    {
        Console.WriteLine($"{Name} says: {Sound}");
    }
    
    public virtual string GetInfo()
    {
        return $"{GetType().Name}: {Name}, Age: {Age}";
    }
    
    public override string ToString() => GetInfo();
}

// Derived class (Child/Sub class)
class Dog : Animal  // Dog สืบทอดจาก Animal
{
    public string Breed { get; set; }
    
    public Dog(string name, int age, string breed)
        : base(name, age)  // เรียก constructor ของ base class
    {
        Breed = breed;
        Sound = "Woof!";
    }
    
    // Override - เขียนทับ method ของ base class
    public override void MakeSound()
    {
        Console.WriteLine($"{Name} barks: {Sound} {Sound}");
    }
    
    public override string GetInfo()
    {
        return base.GetInfo() + $", Breed: {Breed}";
    }
    
    // Method เฉพาะของ Dog
    public void Fetch(string item)
    {
        Console.WriteLine($"{Name} fetches the {item}!");
    }
}

class Cat : Animal
{
    public bool IsIndoor { get; set; }
    
    public Cat(string name, int age, bool isIndoor = true)
        : base(name, age)
    {
        IsIndoor = isIndoor;
        Sound = "Meow~";
    }
    
    public override void MakeSound()
    {
        Console.WriteLine($"{Name} purrs: {Sound}");
    }
    
    public void Purr() => Console.WriteLine($"{Name} is purring...");
}

// การใช้งาน
Animal animal = new Animal("Generic Animal", 5);
Dog dog = new Dog("Max", 3, "Labrador");
Cat cat = new Cat("Whiskers", 2);

animal.MakeSound();  // Generic Animal says:
dog.MakeSound();     // Max barks: Woof! Woof!
cat.MakeSound();     // Whiskers purrs: Meow~

// Polymorphism - ตัวแปร Animal ชี้ไป Dog หรือ Cat ได้
Animal myPet = new Dog("Buddy", 2, "Golden Retriever");
myPet.MakeSound();   // Buddy barks: Woof! Woof! (Dog's method!)

Console.WriteLine(myPet is Dog);   // True
Console.WriteLine(myPet is Cat);   // False
Console.WriteLine(myPet is Animal); // True (Dog IS an Animal)
```

---

## ขั้นตอนที่ 72: Multi-level Inheritance

### Class Hierarchy
```csharp
// Multi-level inheritance
class Vehicle
{
    public string Brand { get; set; }
    public string Model { get; set; }
    public int Year { get; set; }
    public decimal Price { get; set; }
    
    public Vehicle(string brand, string model, int year, decimal price)
    {
        Brand = brand;
        Model = model;
        Year = year;
        Price = price;
    }
    
    public virtual string GetDescription()
    {
        return $"{Year} {Brand} {Model}";
    }
    
    public virtual void Start()
    {
        Console.WriteLine($"{GetDescription()} is starting...");
    }
}

class MotorVehicle : Vehicle
{
    public int HorsePower { get; set; }
    public string FuelType { get; set; }
    public double FuelEfficiency { get; set; }  // km/liter
    
    public MotorVehicle(string brand, string model, int year, decimal price, 
        int hp, string fuelType, double efficiency)
        : base(brand, model, year, price)
    {
        HorsePower = hp;
        FuelType = fuelType;
        FuelEfficiency = efficiency;
    }
    
    public override void Start()
    {
        Console.WriteLine($"{GetDescription()} engine starts: VROOM!");
    }
    
    public double CalculateRange(double fuelTank)
    {
        return fuelTank * FuelEfficiency;
    }
}

class Car : MotorVehicle
{
    public int Doors { get; set; }
    public string BodyStyle { get; set; }  // Sedan, SUV, etc.
    
    public Car(string brand, string model, int year, decimal price,
        int hp, string fuelType, double efficiency, int doors, string body)
        : base(brand, model, year, price, hp, fuelType, efficiency)
    {
        Doors = doors;
        BodyStyle = body;
    }
    
    public override string GetDescription()
    {
        return $"{base.GetDescription()} ({BodyStyle}, {Doors}-door)";
    }
}

class ElectricCar : Car
{
    public double BatteryCapacity { get; set; }  // kWh
    public int ChargingTimeHours { get; set; }
    
    public ElectricCar(string brand, string model, int year, decimal price,
        int hp, double batteryKwh, double rangeKm, int chargingHours, int doors)
        : base(brand, model, year, price, hp, "Electric", rangeKm / batteryKwh, doors, "Sedan")
    {
        BatteryCapacity = batteryKwh;
        ChargingTimeHours = chargingHours;
    }
    
    public override void Start()
    {
        Console.WriteLine($"{GetDescription()} silently starts... (Electric!)");
    }
    
    public override string GetDescription()
    {
        return base.GetDescription() + $" [EV, {BatteryCapacity}kWh]";
    }
}

// การใช้งาน
var tesla = new ElectricCar("Tesla", "Model 3", 2023, 1800000m, 350, 75, 500, 8, 4);
var bmw = new Car("BMW", "330i", 2023, 2500000m, 255, "Petrol", 12.0, 4, "Sedan");
var honda = new MotorVehicle("Honda", "CRF450", 2023, 250000m, 62, "Petrol", 25.0);

tesla.Start();
bmw.Start();
honda.Start();

Console.WriteLine(tesla.GetDescription());
Console.WriteLine($"Range: {tesla.CalculateRange(tesla.BatteryCapacity):F1} km");
Console.WriteLine($"BMW Range: {bmw.CalculateRange(60):F1} km");
```

---

## ขั้นตอนที่ 73: Polymorphism

### Runtime Polymorphism
```csharp
// Polymorphism - ตัวแปรชนิด base type ทำงานแตกต่างกันตาม actual type
List<Animal> animals = new List<Animal>
{
    new Dog("Rex", 3, "German Shepherd"),
    new Cat("Luna", 2),
    new Dog("Molly", 5, "Poodle"),
    new Cat("Tiger", 4, false),
};

// ทุก animal เรียก MakeSound() แต่ทำงานต่างกัน
Console.WriteLine("=== Animal Sounds ===");
foreach (Animal animal in animals)
{
    animal.MakeSound();  // Polymorphic call!
}

// Type checking
foreach (Animal animal in animals)
{
    if (animal is Dog dog)
    {
        dog.Fetch("ball");
    }
    else if (animal is Cat cat)
    {
        cat.Purr();
    }
}

// การใช้กับ Collections
Vehicle[] vehicles = new Vehicle[]
{
    new ElectricCar("Tesla", "Model S", 2023, 3000000m, 400, 100, 600, 6, 4),
    new Car("Toyota", "Camry", 2023, 1200000m, 178, "Hybrid", 18.0, 4, "Sedan"),
    new MotorVehicle("Harley", "Sportster", 2023, 600000m, 65, "Petrol", 20.0)
};

Console.WriteLine("\n=== Vehicle Start ===");
foreach (Vehicle v in vehicles)
{
    v.Start();  // Each behaves differently!
    Console.WriteLine($"  {v.GetDescription()}");
}
```

### Covariance และ Contravariance
```csharp
// Covariance (out) - can use more derived type
IEnumerable<Dog> dogs = new List<Dog>
{
    new Dog("Rex", 3, "Lab"),
    new Dog("Max", 2, "Retriever")
};

IEnumerable<Animal> animals2 = dogs;  // Covariance! Dog เป็น Animal ได้

foreach (Animal a in animals2)
    Console.WriteLine(a.GetInfo());

// Generic covariance
interface IProducer<out T>
{
    T Produce();
}

class DogProducer : IProducer<Dog>
{
    public Dog Produce() => new Dog("NewDog", 0, "Unknown");
}

IProducer<Animal> animalProducer = new DogProducer();  // Works!
Animal produced = animalProducer.Produce();
```

---

## ขั้นตอนที่ 74: Abstract Classes

### Abstract Class
```csharp
// Abstract class - ไม่สามารถสร้าง instance โดยตรงได้
abstract class Shape
{
    public string Color { get; set; } = "Black";
    public string Name { get; protected set; } = "";
    
    // Abstract method - subclass ต้อง implement
    public abstract double CalculateArea();
    public abstract double CalculatePerimeter();
    
    // Virtual method - subclass อาจ override หรือไม่
    public virtual void Draw()
    {
        Console.WriteLine($"Drawing {Name} (Color: {Color})");
        Console.WriteLine($"  Area: {CalculateArea():F2}");
        Console.WriteLine($"  Perimeter: {CalculatePerimeter():F2}");
    }
    
    // Non-virtual method - ไม่สามารถ override
    public string GetDescription()
    {
        return $"{Color} {Name}: Area={CalculateArea():F2}, Perimeter={CalculatePerimeter():F2}";
    }
    
    // Static factory method
    public static Shape CreateCircle(double radius) => new Circle(radius);
    public static Shape CreateRectangle(double w, double h) => new Rectangle(w, h);
}

class Circle : Shape
{
    public double Radius { get; }
    
    public Circle(double radius)
    {
        Radius = radius;
        Name = "Circle";
    }
    
    public override double CalculateArea() => Math.PI * Radius * Radius;
    public override double CalculatePerimeter() => 2 * Math.PI * Radius;
    
    public override void Draw()
    {
        // เรียก base method ก่อน แล้วเพิ่มของตัวเอง
        base.Draw();
        Console.WriteLine($"  Radius: {Radius:F2}");
    }
}

class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }
    
    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
        Name = "Rectangle";
    }
    
    public override double CalculateArea() => Width * Height;
    public override double CalculatePerimeter() => 2 * (Width + Height);
    
    public bool IsSquare => Width == Height;
}

class Triangle : Shape
{
    public double A { get; }
    public double B { get; }
    public double C { get; }
    
    public Triangle(double a, double b, double c)
    {
        if (a + b <= c || a + c <= b || b + c <= a)
            throw new ArgumentException("Invalid triangle sides");
        
        A = a; B = b; C = c;
        Name = "Triangle";
    }
    
    public override double CalculatePerimeter() => A + B + C;
    
    // Heron's formula
    public override double CalculateArea()
    {
        double s = CalculatePerimeter() / 2;
        return Math.Sqrt(s * (s - A) * (s - B) * (s - C));
    }
}

// การใช้งาน Abstract Class
// Shape shape = new Shape();  ← Error! Cannot instantiate abstract class

var shapes = new List<Shape>
{
    new Circle(5.0) { Color = "Red" },
    new Rectangle(4.0, 6.0) { Color = "Blue" },
    new Triangle(3, 4, 5) { Color = "Green" },
    Shape.CreateCircle(3.0)
};

foreach (var shape in shapes)
{
    shape.Draw();
    Console.WriteLine();
}

// สรุป
double totalArea = shapes.Sum(s => s.CalculateArea());
Console.WriteLine($"พื้นที่รวม: {totalArea:F2}");
```

---

## ขั้นตอนที่ 75: sealed Classes และ Methods

### sealed - ป้องกันการสืบทอด
```csharp
// sealed class - ไม่สามารถสืบทอดได้
sealed class FinalConfiguration
{
    public string DatabaseServer { get; init; } = "localhost";
    public int Port { get; init; } = 5432;
    
    private static readonly FinalConfiguration _instance = new();
    public static FinalConfiguration Instance => _instance;
    
    private FinalConfiguration() { }
}

// class ExtendedConfig : FinalConfiguration { }  ← Error!

// sealed method - ป้องกันการ override ในระดับที่ต่ำลงไป
class Base
{
    public virtual void Method() { }
}

class Derived : Base
{
    public sealed override void Method()
    {
        // ถูก override ที่นี่
        // แต่ classes ที่สืบทอดจาก Derived ต่อไปไม่สามารถ override ได้
    }
}

class FurtherDerived : Derived
{
    // public override void Method() { }  ← Error! Method is sealed
}

// Performance benefit ของ sealed
// CLR สามารถ devirtualize calls ไปยัง sealed methods ได้
// ทำให้เร็วกว่า virtual call
```

---

## ขั้นตอนที่ 76: new keyword (Hiding)

### Method Hiding vs Overriding
```csharp
class Parent
{
    public virtual void VirtualMethod()
    {
        Console.WriteLine("Parent.VirtualMethod");
    }
    
    public void NonVirtualMethod()
    {
        Console.WriteLine("Parent.NonVirtualMethod");
    }
}

class Child : Parent
{
    // Override - ใช้ polymorphism
    public override void VirtualMethod()
    {
        Console.WriteLine("Child.VirtualMethod (override)");
    }
    
    // Hide - ซ่อน parent method (ไม่ใช่ polymorphism!)
    public new void NonVirtualMethod()
    {
        Console.WriteLine("Child.NonVirtualMethod (new/hiding)");
    }
}

// ความแตกต่าง:
var child = new Child();
Parent parentRef = child;

child.VirtualMethod();      // Child.VirtualMethod (override)  ← Child
child.NonVirtualMethod();   // Child.NonVirtualMethod (new)    ← Child

parentRef.VirtualMethod();  // Child.VirtualMethod (override)  ← Child (Polymorphism!)
parentRef.NonVirtualMethod(); // Parent.NonVirtualMethod       ← Parent (No polymorphism!)

// ⚠️ Method hiding อาจทำให้สับสน ใช้ด้วยความระวัง
```

---

## ขั้นตอนที่ 77: Object class Methods

### Object.Equals, GetHashCode, ToString
```csharp
class Money : IEquatable<Money>, IComparable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentException("Amount cannot be negative");
        Amount = amount;
        Currency = currency.ToUpper();
    }
    
    // Override Equals
    public override bool Equals(object? obj)
    {
        return Equals(obj as Money);
    }
    
    // IEquatable<Money>
    public bool Equals(Money? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return Amount == other.Amount && Currency == other.Currency;
    }
    
    // Override GetHashCode - ต้อง override เมื่อ override Equals
    public override int GetHashCode()
    {
        return HashCode.Combine(Amount, Currency);
    }
    
    // Override ToString
    public override string ToString()
    {
        return $"{Amount:N2} {Currency}";
    }
    
    // IComparable<Money>
    public int CompareTo(Money? other)
    {
        if (other is null) return 1;
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot compare different currencies");
        return Amount.CompareTo(other.Amount);
    }
    
    // Operators
    public static bool operator ==(Money? left, Money? right)
    {
        if (left is null && right is null) return true;
        if (left is null || right is null) return false;
        return left.Equals(right);
    }
    
    public static bool operator !=(Money? left, Money? right) => !(left == right);
    
    public static bool operator >(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException("Cannot compare different currencies");
        return left.Amount > right.Amount;
    }
    
    public static bool operator <(Money left, Money right) => right > left;
    public static bool operator >=(Money left, Money right) => !(left < right);
    public static bool operator <=(Money left, Money right) => !(left > right);
    
    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(left.Amount + right.Amount, left.Currency);
    }
    
    public static Money operator -(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException("Cannot subtract different currencies");
        return new Money(left.Amount - right.Amount, left.Currency);
    }
    
    public static Money operator *(Money money, decimal multiplier)
    {
        return new Money(money.Amount * multiplier, money.Currency);
    }
}

// การใช้งาน
var m1 = new Money(1000m, "THB");
var m2 = new Money(1000m, "THB");
var m3 = new Money(2000m, "THB");

Console.WriteLine(m1 == m2);        // True
Console.WriteLine(m1 == m3);        // False
Console.WriteLine(m1 < m3);         // True
Console.WriteLine(m1 + m3);         // 3,000.00 THB
Console.WriteLine(m1 * 1.5m);       // 1,500.00 THB

var prices = new List<Money> { m3, m1, m2 };
prices.Sort();
foreach (var p in prices) Console.WriteLine(p);
```

---

## ขั้นตอนที่ 78: Casting และ Type Checking

### Downcasting และ Upcasting
```csharp
// Upcasting - Derived → Base (implicit, always safe)
Dog dog = new Dog("Rex", 3, "Lab");
Animal animal = dog;  // Upcast - OK

// Downcasting - Base → Derived (explicit, may fail)
// 1. Direct cast - throws InvalidCastException if wrong type
Dog dog2 = (Dog)animal;  // OK ถ้า animal จริงๆ เป็น Dog

// 2. as operator - returns null if wrong type
Dog? maybeDog = animal as Dog;    // Dog instance or null
Cat? maybeCat = animal as Cat;    // null (animal is Dog, not Cat)

if (maybeDog != null)
    maybeDog.Fetch("ball");

if (animal as Cat is { } cat)
    cat.Purr();

// 3. Pattern matching (C# 7+) - preferred
if (animal is Dog dogRef)
{
    dogRef.Fetch("stick");
}
else if (animal is Cat catRef)
{
    catRef.Purr();
}

// 4. Switch pattern matching
string message = animal switch
{
    Dog d when d.Breed == "Lab" => $"{d.Name} is a Labrador",
    Dog d => $"{d.Name} is a {d.Breed}",
    Cat c when !c.IsIndoor => $"{c.Name} is an outdoor cat",
    Cat c => $"{c.Name} is an indoor cat",
    _ => "Unknown animal"
};
Console.WriteLine(message);

// GetType() vs typeof()
Console.WriteLine(animal.GetType() == typeof(Dog));    // True (runtime type)
Console.WriteLine(animal.GetType() == typeof(Animal)); // False
Console.WriteLine(animal is Animal);                   // True (includes derived)
```

---

## ขั้นตอนที่ 79: Extension ของ Inheritance

### Multiple Inheritance ใน C#
```csharp
// C# ไม่รองรับ multiple inheritance สำหรับ classes
// แต่รองรับ multiple interface implementation

interface IFlyable
{
    double MaxAltitude { get; }
    void Fly();
}

interface ISwimmable
{
    double MaxDepth { get; }
    void Swim();
}

interface IDivable : ISwimmable
{
    void Dive(double depth);
}

// Class สามารถ implement หลาย interfaces
class Duck : Animal, IFlyable, ISwimmable
{
    public double MaxAltitude => 100;
    public double MaxDepth => 0.5;
    
    public Duck(string name, int age) 
        : base(name, age) 
    { 
        Sound = "Quack!";
    }
    
    public void Fly()
    {
        Console.WriteLine($"{Name} is flying at max {MaxAltitude}m");
    }
    
    public void Swim()
    {
        Console.WriteLine($"{Name} is swimming!");
    }
    
    public override void MakeSound()
    {
        Console.WriteLine($"{Name}: {Sound} {Sound}");
    }
}

// ใช้ polymorphism กับ interfaces
List<IFlyable> flyingThings = new()
{
    new Duck("Donald", 3),
    // new Airplane("Boeing 747") ถ้ามี class Airplane : IFlyable
};

foreach (var thing in flyingThings)
    thing.Fly();
```

---

## ขั้นตอนที่ 80: โปรแกรมตัวอย่าง - Employee Hierarchy

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace EmployeeSystem
{
    enum EmployeeType { FullTime, PartTime, Contractor, Intern }
    
    abstract class Employee
    {
        private static int _nextId = 1;
        
        public int Id { get; } = _nextId++;
        public string Name { get; set; }
        public string Department { get; set; }
        public DateTime HireDate { get; set; }
        public EmployeeType Type { get; protected set; }
        
        protected Employee(string name, string department)
        {
            Name = name;
            Department = department;
            HireDate = DateTime.Now;
        }
        
        // Abstract - subclass must implement
        public abstract decimal CalculateSalary();
        public abstract string GetEmployeeType();
        
        // Virtual - subclass can override
        public virtual string GetDetails()
        {
            return $"[{Id}] {Name} - {GetEmployeeType()} in {Department}";
        }
        
        public int YearsOfService => (int)((DateTime.Now - HireDate).Days / 365.25);
        
        public override string ToString() => GetDetails();
    }
    
    class FullTimeEmployee : Employee
    {
        public decimal BaseSalary { get; set; }
        public decimal BonusPercentage { get; set; } = 0;
        public List<string> Benefits { get; } = new() { "ประกันสุขภาพ", "กองทุนสำรอง" };
        
        public FullTimeEmployee(string name, string department, decimal salary)
            : base(name, department)
        {
            BaseSalary = salary;
            Type = EmployeeType.FullTime;
        }
        
        public override decimal CalculateSalary()
        {
            decimal bonus = BaseSalary * BonusPercentage / 100;
            return BaseSalary + bonus;
        }
        
        public override string GetEmployeeType() => "พนักงานประจำ";
        
        public override string GetDetails()
        {
            return base.GetDetails() + 
                $"\n  เงินเดือน: {BaseSalary:N2} บาท" +
                $"\n  โบนัส: {BonusPercentage}%";
        }
    }
    
    class Manager : FullTimeEmployee
    {
        public List<Employee> Team { get; } = new();
        public decimal ManagementAllowance { get; set; } = 10000m;
        
        public Manager(string name, string department, decimal salary)
            : base(name, department, salary)
        {
            Type = EmployeeType.FullTime;
        }
        
        public override decimal CalculateSalary()
        {
            return base.CalculateSalary() + ManagementAllowance;
        }
        
        public override string GetEmployeeType() => "ผู้จัดการ";
        
        public void AddTeamMember(Employee employee)
        {
            if (!Team.Contains(employee))
                Team.Add(employee);
        }
        
        public decimal GetTeamSalaryBudget()
        {
            return Team.Sum(e => e.CalculateSalary()) + CalculateSalary();
        }
        
        public override string GetDetails()
        {
            return base.GetDetails() + 
                $"\n  ค่าบริหาร: {ManagementAllowance:N2} บาท" +
                $"\n  ทีม: {Team.Count} คน";
        }
    }
    
    class PartTimeEmployee : Employee
    {
        public decimal HourlyRate { get; set; }
        public int HoursPerWeek { get; set; }
        public int WeeksPerMonth { get; set; } = 4;
        
        public PartTimeEmployee(string name, string department, decimal hourlyRate, int hoursPerWeek)
            : base(name, department)
        {
            HourlyRate = hourlyRate;
            HoursPerWeek = hoursPerWeek;
            Type = EmployeeType.PartTime;
        }
        
        public override decimal CalculateSalary()
        {
            return HourlyRate * HoursPerWeek * WeeksPerMonth;
        }
        
        public override string GetEmployeeType() => "พนักงานนอกเวลา";
        
        public override string GetDetails()
        {
            return base.GetDetails() + 
                $"\n  อัตราชั่วโมง: {HourlyRate:N2} บาท" +
                $"\n  ชั่วโมง/สัปดาห์: {HoursPerWeek}";
        }
    }
    
    class Contractor : Employee
    {
        public decimal DailyRate { get; set; }
        public int WorkDaysPerMonth { get; set; } = 22;
        public DateTime ContractEndDate { get; set; }
        
        public Contractor(string name, string department, decimal dailyRate, DateTime endDate)
            : base(name, department)
        {
            DailyRate = dailyRate;
            ContractEndDate = endDate;
            Type = EmployeeType.Contractor;
        }
        
        public bool IsContractExpired => DateTime.Now > ContractEndDate;
        
        public int DaysRemainingInContract => 
            Math.Max(0, (int)(ContractEndDate - DateTime.Now).TotalDays);
        
        public override decimal CalculateSalary()
        {
            if (IsContractExpired) return 0;
            return DailyRate * WorkDaysPerMonth;
        }
        
        public override string GetEmployeeType() => "ผู้รับจ้างภายนอก";
        
        public override string GetDetails()
        {
            return base.GetDetails() + 
                $"\n  อัตราวัน: {DailyRate:N2} บาท" +
                $"\n  สัญญาหมด: {ContractEndDate:dd/MM/yyyy}" +
                $"\n  เหลือ: {DaysRemainingInContract} วัน";
        }
    }
    
    class Intern : PartTimeEmployee
    {
        public string University { get; set; }
        public string Program { get; set; }
        public DateTime InternshipEnd { get; set; }
        
        public Intern(string name, string department, decimal stipend, string university, string program)
            : base(name, department, stipend / 20, 20)  // ค่าตอบแทน / วัน → hourly rate
        {
            University = university;
            Program = program;
            InternshipEnd = DateTime.Now.AddMonths(3);
            Type = EmployeeType.Intern;
        }
        
        public override string GetEmployeeType() => "นักศึกษาฝึกงาน";
        
        public override string GetDetails()
        {
            return base.GetDetails() + 
                $"\n  มหาวิทยาลัย: {University}" +
                $"\n  โปรแกรม: {Program}" +
                $"\n  ฝึกงานถึง: {InternshipEnd:dd/MM/yyyy}";
        }
    }
    
    class Company
    {
        public string Name { get; }
        private List<Employee> _employees = new();
        
        public Company(string name) => Name = name;
        
        public T Add<T>(T employee) where T : Employee
        {
            _employees.Add(employee);
            return employee;
        }
        
        public decimal GetTotalPayroll() => _employees.Sum(e => e.CalculateSalary());
        
        public Dictionary<string, decimal> GetPayrollByDepartment()
        {
            return _employees
                .GroupBy(e => e.Department)
                .ToDictionary(g => g.Key, g => g.Sum(e => e.CalculateSalary()));
        }
        
        public Dictionary<EmployeeType, int> GetHeadcountByType()
        {
            return _employees
                .GroupBy(e => e.Type)
                .ToDictionary(g => g.Key, g => g.Count());
        }
        
        public void PrintPayrollReport()
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine($"\n╔══════════════════════════════════════════╗");
            Console.WriteLine($"║       รายงานเงินเดือน: {Name,-19}║");
            Console.WriteLine($"╚══════════════════════════════════════════╝");
            Console.ResetColor();
            
            Console.WriteLine($"\n{"ชื่อ",-25} {"ประเภท",-20} {"เงินเดือน",12}");
            Console.WriteLine(new string('─', 60));
            
            foreach (var emp in _employees.OrderBy(e => e.Department).ThenBy(e => e.Name))
            {
                decimal salary = emp.CalculateSalary();
                Console.ForegroundColor = emp.Type switch
                {
                    EmployeeType.FullTime => ConsoleColor.White,
                    EmployeeType.PartTime => ConsoleColor.Yellow,
                    EmployeeType.Contractor => ConsoleColor.Cyan,
                    EmployeeType.Intern => ConsoleColor.Green,
                    _ => ConsoleColor.Gray
                };
                Console.WriteLine($"{emp.Name,-25} {emp.GetEmployeeType(),-20} {salary,12:N2}");
                Console.ResetColor();
            }
            
            Console.WriteLine(new string('─', 60));
            Console.WriteLine($"{"ยอดรวม",-45} {GetTotalPayroll(),12:N2}");
            
            Console.WriteLine("\n--- เงินเดือนตามแผนก ---");
            foreach (var (dept, total) in GetPayrollByDepartment().OrderByDescending(x => x.Value))
                Console.WriteLine($"{dept}: {total:N2} บาท");
            
            Console.WriteLine("\n--- จำนวนพนักงานตามประเภท ---");
            foreach (var (type, count) in GetHeadcountByType())
                Console.WriteLine($"{type}: {count} คน");
        }
    }
    
    class Program
    {
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            var company = new Company("บริษัท ABC จำกัด");
            
            // เพิ่มพนักงาน
            var cto = company.Add(new Manager("สมชาย วิชาการ", "IT", 120000m) 
                { BonusPercentage = 20 });
            var dev1 = company.Add(new FullTimeEmployee("อลิซ", "IT", 65000m) 
                { BonusPercentage = 10 });
            var dev2 = company.Add(new FullTimeEmployee("บ็อบ", "IT", 60000m));
            var contractor = company.Add(new Contractor("ชาร์ลี", "IT", 3500m, 
                DateTime.Now.AddMonths(6)));
            var intern = company.Add(new Intern("ไดอาน่า", "IT", 15000m, 
                "จุฬาลงกรณ์", "วิทยาการคอมพิวเตอร์"));
            var hrManager = company.Add(new Manager("สมหญิง ใจดี", "HR", 100000m));
            var hrStaff = company.Add(new PartTimeEmployee("อีฟ", "HR", 200m, 20));
            var salesManager = company.Add(new Manager("วิชาญ โปร", "Sales", 110000m) 
                { BonusPercentage = 15 });
            
            // จัด team
            cto.AddTeamMember(dev1);
            cto.AddTeamMember(dev2);
            cto.AddTeamMember(contractor);
            cto.AddTeamMember(intern);
            
            // แสดงรายงาน
            company.PrintPayrollReport();
            
            // แสดงรายละเอียดผู้จัดการ
            Console.WriteLine($"\n--- รายละเอียด Manager ---");
            Console.WriteLine(cto.GetDetails());
            Console.WriteLine($"\nงบประมาณทีม IT: {cto.GetTeamSalaryBudget():N2} บาท");
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 08

| หัวข้อ | รายละเอียดสำคัญ |
|--------|----------------|
| Inheritance | `: BaseClass` สืบทอด fields, properties, methods |
| base keyword | เรียก constructor/method ของ base class |
| virtual | Method สามารถถูก override ได้ |
| override | เขียนทับ virtual method |
| abstract | Method ที่ต้อง implement, ไม่มี body |
| sealed | ป้องกันการสืบทอด หรือ override |
| new (hiding) | ซ่อน base method (ไม่ใช่ polymorphism!) |
| Polymorphism | ตัวแปร base type ทำงานตาม actual type |
| Upcasting | Derived → Base (implicit, safe) |
| Downcasting | Base → Derived (explicit, may throw) |

---

## 🏋️ แบบฝึกหัด Part 08

### แบบฝึกหัดที่ 1: Shape Calculator
สร้าง hierarchy: Shape → 2DShape, 3DShape → Circle, Square, Cube, Sphere
- abstract CalculateArea(), CalculateVolume() สำหรับ 3D

### แบบฝึกหัดที่ 2: Payment System
สร้าง abstract class Payment → CreditCard, BankTransfer, DigitalWallet
- abstract ProcessPayment(), ValidatePayment()
- แต่ละชนิดมี logic ต่างกัน

### แบบฝึกหัดที่ 3: Vehicle Rental
สร้าง hierarchy Vehicle → Car, Truck, Motorcycle
- CalculateRentalCost(int days) ต่างกันแต่ละประเภท
- abstract method GetVehicleType()

---

**ก่อนหน้า → [Part 07: OOP Classes](part07-oop-classes.md)**  
**ต่อไป → [Part 09: Interfaces & Abstract Classes](part09-interfaces.md)**
