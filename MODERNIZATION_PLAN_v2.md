# GestionCabinet Modernization Plan v2.0 (Enhanced with Cabinet_Medical Insights)

## Executive Summary
Transform GestionCabinet from a Thymeleaf MVC application into a modern full-stack system, **inspired by Cabinet_Medical's proven architecture**, with:
- **Frontend**: React with TypeScript (replacing Thymeleaf)
- **Backend**: Spring Boot REST API (refactored with JPA + Hibernate patterns)
- **New Features**: Doctor management, advanced appointment scheduling, prescriptions, medical file system
- **Architecture**: Decoupled backend/frontend, inspired by Cabinet_Medical's well-tested DAO pattern but modernized with ORM

---

## Key Insights from Cabinet_Medical Project

Cabinet_Medical (JAVA EE) demonstrates proven patterns we'll modernize:

### ✅ What to Keep & Modernize
```
OLD (Cabinet_Medical - JAVA EE):          NEW (GestionCabinet 2.0 - Spring Boot):
- Servlet-based routing                   → Spring Rest Controllers
- DAO Layer with manual SQL               → Spring Data JPA (with query methods)
- JSP + HTML/CSS frontend                 → React TypeScript SPA
- Plain JDBC connections                  → Hibernate ORM + Connection Pooling
- Session-based auth                      → JWT + Spring Security
- Manual appointment conflict handling    → Database constraints + business logic
- Prescription management (manual)        → Prescription entity with lifecycle

PROVEN FEATURES TO INCLUDE:
✓ Appointment scheduling with conflict detection (Cabinet_Medical has this!)
✓ Prescription management system
✓ Medical file with consultation history
✓ Doctor/Patient role separation
✓ Notification system for appointments
✓ Dashboard with statistics
✓ User authentication with role-based access
```

### 📊 Enhanced Data Model (Inspired by Cabinet_Medical)

**Existing Cabinet_Medical Entities (to adapt):**
- User (base class) - Doctor & Patient extend this
- Patient (birthDate, sex, medical file)
- Doctor (specialty)
- Appointment (date, description, illness type, notification flag)
- Prescription
- Consultation
- Medical File

**New/Enhanced for GestionCabinet:**
- Diagnosis records with treatment tracking
- Appointment status workflow (Cabinet_Medical uses notification flag, we'll enhance with status)
- Doctor availability/schedule management
- Statistics & analytics aggregation

---

## Phase 1: Backend Refactoring & Entity Modernization (Week 1-3)

### 1.1 Update Database Schema

```sql
-- Users base table (with roles)
CREATE TABLE users (
    id_user INT PRIMARY KEY AUTO_INCREMENT,
    cin VARCHAR(20) UNIQUE NOT NULL,
    firstName VARCHAR(50) NOT NULL,
    lastName VARCHAR(50) NOT NULL,
    phone VARCHAR(20) UNIQUE,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    account_type ENUM('DOCTOR', 'PATIENT', 'ADMIN', 'RECEPTIONIST'),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Patients table (extends users)
CREATE TABLE patients (
    id_patient INT PRIMARY KEY AUTO_INCREMENT,
    id_user INT NOT NULL UNIQUE,
    birthDate DATE,
    sex ENUM('MALE', 'FEMALE', 'OTHER'),
    blood_type VARCHAR(10),
    allergies TEXT,
    medical_conditions TEXT,
    insurance_number VARCHAR(50),
    emergency_contact VARCHAR(100),
    FOREIGN KEY (id_user) REFERENCES users(id_user) ON DELETE CASCADE
);

-- Doctors table (extends users)
CREATE TABLE doctors (
    id_doctor INT PRIMARY KEY AUTO_INCREMENT,
    id_user INT NOT NULL UNIQUE,
    specialty VARCHAR(100) NOT NULL,
    license_number VARCHAR(50) UNIQUE NOT NULL,
    office_phone VARCHAR(20),
    office_address VARCHAR(255),
    years_of_experience INT,
    is_available BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_user) REFERENCES users(id_user) ON DELETE CASCADE
);

-- Doctor availability/schedule
CREATE TABLE doctor_schedules (
    id_schedule INT PRIMARY KEY AUTO_INCREMENT,
    id_doctor INT NOT NULL,
    day_of_week INT, -- 0=Monday, 6=Sunday
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    slot_duration INT DEFAULT 30, -- in minutes
    FOREIGN KEY (id_doctor) REFERENCES doctors(id_doctor) ON DELETE CASCADE,
    INDEX (id_doctor)
);

-- Appointments (inspired by Cabinet_Medical but enhanced)
CREATE TABLE appointments (
    id_appointment INT PRIMARY KEY AUTO_INCREMENT,
    id_patient INT NOT NULL,
    id_doctor INT,
    date_of_appointment DATETIME NOT NULL,
    date_of_checking DATETIME,
    type_of_illness VARCHAR(100),
    description TEXT,
    status ENUM('SCHEDULED', 'IN_PROGRESS', 'COMPLETED', 'CANCELLED', 'NO_SHOW') DEFAULT 'SCHEDULED',
    notification BOOLEAN DEFAULT FALSE,
    notification_sent_at DATETIME,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (id_patient) REFERENCES patients(id_patient) ON DELETE CASCADE,
    FOREIGN KEY (id_doctor) REFERENCES doctors(id_doctor) ON DELETE SET NULL,
    INDEX (id_patient),
    INDEX (id_doctor),
    INDEX (date_of_appointment),
    INDEX (status)
);

-- Consultations (medical records from appointments)
CREATE TABLE consultations (
    id_consultation INT PRIMARY KEY AUTO_INCREMENT,
    id_appointment INT NOT NULL,
    id_patient INT NOT NULL,
    id_doctor INT,
    date DATETIME NOT NULL,
    description TEXT,
    diagnosis TEXT,
    treatment_plan TEXT,
    follow_up_date DATETIME,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_appointment) REFERENCES appointments(id_appointment) ON DELETE CASCADE,
    FOREIGN KEY (id_patient) REFERENCES patients(id_patient) ON DELETE CASCADE,
    FOREIGN KEY (id_doctor) REFERENCES doctors(id_doctor) ON DELETE SET NULL,
    INDEX (id_patient),
    INDEX (id_doctor)
);

-- Prescriptions (new)
CREATE TABLE prescriptions (
    id_prescription INT PRIMARY KEY AUTO_INCREMENT,
    id_consultation INT NOT NULL,
    id_patient INT NOT NULL,
    id_doctor INT,
    medication_name VARCHAR(100) NOT NULL,
    dosage VARCHAR(100) NOT NULL,
    frequency VARCHAR(100) NOT NULL,
    duration_days INT,
    instructions TEXT,
    date_prescribed DATETIME DEFAULT CURRENT_TIMESTAMP,
    date_dispensed DATETIME,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (id_consultation) REFERENCES consultations(id_consultation) ON DELETE CASCADE,
    FOREIGN KEY (id_patient) REFERENCES patients(id_patient) ON DELETE CASCADE,
    FOREIGN KEY (id_doctor) REFERENCES doctors(id_doctor) ON DELETE SET NULL,
    INDEX (id_patient)
);

-- Medical File (aggregate of patient medical records)
CREATE TABLE medical_files (
    id_medical_file INT PRIMARY KEY AUTO_INCREMENT,
    id_patient INT NOT NULL UNIQUE,
    total_consultations INT DEFAULT 0,
    last_consultation_date DATETIME,
    blood_type VARCHAR(10),
    chronic_diseases TEXT,
    surgeries TEXT,
    vaccinations TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (id_patient) REFERENCES patients(id_patient) ON DELETE CASCADE
);

-- Diagnosis records (enhanced tracking)
CREATE TABLE diagnoses (
    id_diagnosis INT PRIMARY KEY AUTO_INCREMENT,
    id_consultation INT NOT NULL,
    diagnostic_code VARCHAR(20),
    diagnostic_description TEXT NOT NULL,
    severity ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL'),
    date_diagnosed DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_consultation) REFERENCES consultations(id_consultation) ON DELETE CASCADE
);
```

### 1.2 Create Entity Classes (JPA)

```java
// User (Base Entity)
@Entity
@Table(name = "users")
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "account_type")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_user;
    
    @Column(unique = true, length = 20)
    private String cin;
    
    @Column(length = 50)
    private String firstName;
    
    @Column(length = 50)
    private String lastName;
    
    @Column(unique = true, length = 20)
    private String phone;
    
    @Column(unique = true, length = 100)
    private String email;
    
    @Column(length = 255)
    private String password;
    
    @Enumerated(EnumType.STRING)
    private AccountType account_type;
    
    @CreationTimestamp
    private LocalDateTime created_at;
    
    @UpdateTimestamp
    private LocalDateTime updated_at;
}

// Patient Entity
@Entity
@Table(name = "patients")
public class Patient {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_patient;
    
    @OneToOne
    @JoinColumn(name = "id_user")
    private User user;
    
    @Column(name = "birthDate")
    private LocalDate birthDate;
    
    @Enumerated(EnumType.STRING)
    private Gender sex;
    
    private String blood_type;
    private String allergies;
    private String medical_conditions;
    private String insurance_number;
    private String emergency_contact;
    
    @OneToMany(mappedBy = "patient", cascade = CascadeType.ALL)
    private List<Appointment> appointments;
    
    @OneToOne(mappedBy = "patient", cascade = CascadeType.ALL)
    private MedicalFile medicalFile;
}

// Doctor Entity
@Entity
@Table(name = "doctors")
public class Doctor {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_doctor;
    
    @OneToOne
    @JoinColumn(name = "id_user")
    private User user;
    
    @Column(nullable = false, length = 100)
    private String specialty;
    
    @Column(unique = true, length = 50)
    private String license_number;
    
    private String office_phone;
    private String office_address;
    private Integer years_of_experience;
    private Boolean is_available = true;
    
    @OneToMany(mappedBy = "doctor", cascade = CascadeType.ALL)
    private List<Appointment> appointments;
    
    @OneToMany(mappedBy = "doctor", cascade = CascadeType.ALL)
    private List<DoctorSchedule> schedules;
}

// Appointment Entity (inspired by Cabinet_Medical + enhanced)
@Entity
@Table(name = "appointments")
public class Appointment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_appointment;
    
    @ManyToOne
    @JoinColumn(name = "id_patient")
    private Patient patient;
    
    @ManyToOne
    @JoinColumn(name = "id_doctor")
    private Doctor doctor;
    
    @Column(name = "date_of_appointment", nullable = false)
    private LocalDateTime dateofAppointment;
    
    @Column(name = "date_of_checking")
    private LocalDateTime dateofChecking;
    
    @Column(name = "type_of_illness")
    private String typeofIllness;
    
    @Column(name = "description")
    private String description;
    
    @Enumerated(EnumType.STRING)
    private AppointmentStatus status = AppointmentStatus.SCHEDULED;
    
    private Boolean notification = false;
    private LocalDateTime notification_sent_at;
    private String notes;
    
    @CreationTimestamp
    private LocalDateTime created_at;
    
    @UpdateTimestamp
    private LocalDateTime updated_at;
}

// Prescription Entity
@Entity
@Table(name = "prescriptions")
public class Prescription {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_prescription;
    
    @ManyToOne
    @JoinColumn(name = "id_consultation")
    private Consultation consultation;
    
    @ManyToOne
    @JoinColumn(name = "id_patient")
    private Patient patient;
    
    @ManyToOne
    @JoinColumn(name = "id_doctor")
    private Doctor doctor;
    
    @Column(nullable = false)
    private String medication_name;
    
    @Column(nullable = false)
    private String dosage;
    
    @Column(nullable = false)
    private String frequency;
    
    private Integer duration_days;
    private String instructions;
    
    @CreationTimestamp
    private LocalDateTime date_prescribed;
    
    private LocalDateTime date_dispensed;
    private Boolean is_active = true;
}

// Medical File Entity
@Entity
@Table(name = "medical_files")
public class MedicalFile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_medical_file;
    
    @OneToOne
    @JoinColumn(name = "id_patient")
    private Patient patient;
    
    private Integer total_consultations = 0;
    private LocalDateTime last_consultation_date;
    private String blood_type;
    private String chronic_diseases;
    private String surgeries;
    private String vaccinations;
    
    @UpdateTimestamp
    private LocalDateTime updated_at;
}

// Doctor Schedule Entity
@Entity
@Table(name = "doctor_schedules")
public class DoctorSchedule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_schedule;
    
    @ManyToOne
    @JoinColumn(name = "id_doctor")
    private Doctor doctor;
    
    private Integer day_of_week; // 0=Monday, 6=Sunday
    private LocalTime start_time;
    private LocalTime end_time;
    private Integer slot_duration = 30; // minutes
}

// Consultation Entity
@Entity
@Table(name = "consultations")
public class Consultation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id_consultation;
    
    @ManyToOne
    @JoinColumn(name = "id_appointment")
    private Appointment appointment;
    
    @ManyToOne
    @JoinColumn(name = "id_patient")
    private Patient patient;
    
    @ManyToOne
    @JoinColumn(name = "id_doctor")
    private Doctor doctor;
    
    @Column(nullable = false)
    private LocalDateTime date;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(columnDefinition = "TEXT")
    private String diagnosis;
    
    @Column(columnDefinition = "TEXT")
    private String treatment_plan;
    
    private LocalDateTime follow_up_date;
    
    @OneToMany(mappedBy = "consultation", cascade = CascadeType.ALL)
    private List<Prescription> prescriptions;
    
    @CreationTimestamp
    private LocalDateTime created_at;
}
```

### 1.3 Create Repositories (Spring Data JPA)

```java
// PatientRepository
@Repository
public interface PatientRepository extends JpaRepository<Patient, Long> {
    Page<Patient> findByUser_FirstNameContainingIgnoreCaseOrUser_LastNameContainingIgnoreCase(
        String firstName, String lastName, Pageable pageable
    );
    Optional<Patient> findByUser_Email(String email);
    List<Patient> findByBirthDateBetween(LocalDate start, LocalDate end);
}

// DoctorRepository
@Repository
public interface DoctorRepository extends JpaRepository<Doctor, Long> {
    Page<Doctor> findBySpecialtyContainingIgnoreCase(String specialty, Pageable pageable);
    Optional<Doctor> findByUser_Email(String email);
    List<Doctor> findBySpecialtyAndIs_availableTrue(String specialty);
}

// AppointmentRepository
@Repository
public interface AppointmentRepository extends JpaRepository<Appointment, Long> {
    List<Appointment> findByPatient_Id_patient(Long patientId);
    List<Appointment> findByDoctor_Id_doctor(Long doctorId);
    List<Appointment> findByDateofAppointmentBetween(LocalDateTime start, LocalDateTime end);
    List<Appointment> findByStatus(AppointmentStatus status);
    List<Appointment> findByStatusAndDateofAppointmentGreaterThan(AppointmentStatus status, LocalDateTime date);
    // For Cabinet_Medical inspired filtering
    List<Appointment> findByDateofAppointmentGreaterThanAndNotificationFalse(LocalDateTime date);
    List<Appointment> findByDateofAppointmentLessThan(LocalDateTime date);
}

// ConsultationRepository
@Repository
public interface ConsultationRepository extends JpaRepository<Consultation, Long> {
    List<Consultation> findByPatient_Id_patient(Long patientId);
    List<Consultation> findByDoctor_Id_doctor(Long doctorId);
    List<Consultation> findByDateBetween(LocalDateTime start, LocalDateTime end);
}

// PrescriptionRepository
@Repository
public interface PrescriptionRepository extends JpaRepository<Prescription, Long> {
    List<Prescription> findByPatient_Id_patientAndIs_activeTrue(Long patientId);
    List<Prescription> findByConsultation_Id_consultation(Long consultationId);
    List<Prescription> findByDoctor_Id_doctor(Long doctorId);
}

// UserRepository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    Optional<User> findByCin(String cin);
}
```

### 1.4 Create Service Layer

```java
// AppointmentService (inspired by Cabinet_Medical's business logic)
@Service
@Transactional
public class AppointmentService {
    @Autowired
    private AppointmentRepository appointmentRepository;
    
    @Autowired
    private DoctorRepository doctorRepository;
    
    @Autowired
    private PatientRepository patientRepository;
    
    // Get list of upcoming appointments (Cabinet_Medical: ListeAppointmentNF)
    public List<AppointmentDTO> getUpcomingAppointments() {
        return appointmentRepository
            .findByDateofAppointmentGreaterThanAndNotificationFalse(LocalDateTime.now())
            .stream()
            .map(this::convertToDTO)
            .collect(Collectors.toList());
    }
    
    // Get list of finished appointments (Cabinet_Medical: ListeAppointmentF)
    public List<AppointmentDTO> getFinishedAppointments() {
        return appointmentRepository
            .findByDateofAppointmentLessThan(LocalDateTime.now())
            .stream()
            .map(this::convertToDTO)
            .collect(Collectors.toList());
    }
    
    // Check appointment conflict (Cabinet_Medical: getAppointmentByDate)
    public boolean checkConflict(Long doctorId, LocalDateTime appointmentDate, int durationMinutes) {
        LocalDateTime endTime = appointmentDate.plusMinutes(durationMinutes);
        List<Appointment> existingAppointments = appointmentRepository
            .findByDoctor_Id_doctor(doctorId).stream()
            .filter(a -> a.getStatus() != AppointmentStatus.CANCELLED)
            .filter(a -> hasTimeConflict(a, appointmentDate, endTime))
            .collect(Collectors.toList());
        return !existingAppointments.isEmpty();
    }
    
    private boolean hasTimeConflict(Appointment existing, LocalDateTime start, LocalDateTime end) {
        return !(existing.getDateofAppointment().isAfter(end) || 
                 existing.getDateofAppointment().isBefore(start));
    }
    
    // Take appointment (Cabinet_Medical: takeAppointment)
    public AppointmentDTO takeAppointment(CreateAppointmentRequest request) {
        if (checkConflict(request.getDoctorId(), request.getAppointmentDate(), 30)) {
            throw new ConflictException("This time slot is not available");
        }
        
        Patient patient = patientRepository.findById(request.getPatientId())
            .orElseThrow(() -> new ResourceNotFoundException("Patient not found"));
        Doctor doctor = doctorRepository.findById(request.getDoctorId())
            .orElseThrow(() -> new ResourceNotFoundException("Doctor not found"));
        
        Appointment appointment = new Appointment();
        appointment.setPatient(patient);
        appointment.setDoctor(doctor);
        appointment.setDateofAppointment(request.getAppointmentDate());
        appointment.setTypeofIllness(request.getTypeofIllness());
        appointment.setDescription(request.getDescription());
        appointment.setStatus(AppointmentStatus.SCHEDULED);
        appointment.setNotification(false);
        
        return convertToDTO(appointmentRepository.save(appointment));
    }
    
    // Update notification (Cabinet_Medical: Updatenotification)
    public void markAppointmentsNotified() {
        List<Appointment> unnotified = appointmentRepository
            .findByDateofAppointmentGreaterThanAndNotificationFalse(LocalDateTime.now());
        unnotified.forEach(a -> {
            a.setNotification(true);
            a.setNotification_sent_at(LocalDateTime.now());
        });
        appointmentRepository.saveAll(unnotified);
    }
    
    // Get notification count
    public int getUnnotifiedAppointmentCount() {
        return (int) appointmentRepository.findAll().stream()
            .filter(a -> !a.getNotification() && a.getDateofAppointment().isAfter(LocalDateTime.now()))
            .count();
    }
    
    private AppointmentDTO convertToDTO(Appointment appointment) {
        // Conversion logic
        return new AppointmentDTO(appointment);
    }
}

// PrescriptionService (new)
@Service
@Transactional
public class PrescriptionService {
    @Autowired
    private PrescriptionRepository prescriptionRepository;
    
    public List<PrescriptionDTO> getActivePrescriptionsForPatient(Long patientId) {
        return prescriptionRepository.findByPatient_Id_patientAndIs_activeTrue(patientId)
            .stream()
            .map(PrescriptionDTO::new)
            .collect(Collectors.toList());
    }
    
    public PrescriptionDTO createPrescription(CreatePrescriptionRequest request) {
        Prescription prescription = new Prescription();
        prescription.setMedication_name(request.getMedicationName());
        prescription.setDosage(request.getDosage());
        prescription.setFrequency(request.getFrequency());
        prescription.setDuration_days(request.getDurationDays());
        prescription.setInstructions(request.getInstructions());
        prescription.setIs_active(true);
        
        return new PrescriptionDTO(prescriptionRepository.save(prescription));
    }
}

// StatisticsService (new - enhanced dashboard)
@Service
public class StatisticsService {
    @Autowired
    private AppointmentRepository appointmentRepository;
    
    @Autowired
    private PatientRepository patientRepository;
    
    @Autowired
    private DoctorRepository doctorRepository;
    
    @Autowired
    private ConsultationRepository consultationRepository;
    
    public DashboardStatsDTO getDashboardStats() {
        LocalDateTime monthStart = LocalDateTime.now().withDayOfMonth(1).toLocalDate().atStartOfDay();
        
        long consultationsThisMonth = appointmentRepository
            .findByDateofAppointmentBetween(monthStart, LocalDateTime.now())
            .size();
        
        return DashboardStatsDTO.builder()
            .totalPatients(patientRepository.count())
            .totalDoctors(doctorRepository.count())
            .totalConsultations(consultationRepository.count())
            .consultationsThisMonth(consultationsThisMonth)
            .build();
    }
    
    public List<ConsultationTrendDTO> getMonthlyTrends(int months) {
        // Aggregation logic for trends
        return new ArrayList<>();
    }
    
    public Map<String, Long> getSpecialityDistribution() {
        return doctorRepository.findAll().stream()
            .collect(Collectors.groupingByConcurrent(Doctor::getSpecialty, Collectors.counting()));
    }
}
```

### 1.5 Create REST Controllers

```java
// AppointmentController (REST API)
@RestController
@RequestMapping("/api/appointments")
@CrossOrigin(origins = "${cors.allowed-origins}")
public class AppointmentController {
    @Autowired
    private AppointmentService appointmentService;
    
    @GetMapping
    public ResponseEntity<List<AppointmentDTO>> getAllAppointments(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(appointmentService.getUpcomingAppointments());
    }
    
    @GetMapping("/upcoming")
    public ResponseEntity<List<AppointmentDTO>> getUpcomingAppointments() {
        return ResponseEntity.ok(appointmentService.getUpcomingAppointments());
    }
    
    @GetMapping("/finished")
    public ResponseEntity<List<AppointmentDTO>> getFinishedAppointments() {
        return ResponseEntity.ok(appointmentService.getFinishedAppointments());
    }
    
    @PostMapping
    public ResponseEntity<AppointmentDTO> createAppointment(@RequestBody CreateAppointmentRequest request) {
        return ResponseEntity.status(201).body(appointmentService.takeAppointment(request));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<AppointmentDTO> updateAppointment(@PathVariable Long id, 
                                                            @RequestBody UpdateAppointmentRequest request) {
        return ResponseEntity.ok(appointmentService.updateAppointment(id, request));
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteAppointment(@PathVariable Long id) {
        appointmentService.deleteAppointment(id);
        return ResponseEntity.noContent().build();
    }
}

// StatisticsController
@RestController
@RequestMapping("/api/statistics")
@CrossOrigin(origins = "${cors.allowed-origins}")
public class StatisticsController {
    @Autowired
    private StatisticsService statisticsService;
    
    @GetMapping("/dashboard")
    public ResponseEntity<DashboardStatsDTO> getDashboardStats() {
        return ResponseEntity.ok(statisticsService.getDashboardStats());
    }
    
    @GetMapping("/trends/monthly")
    public ResponseEntity<List<ConsultationTrendDTO>> getMonthlyTrends(
            @RequestParam(defaultValue = "12") int months) {
        return ResponseEntity.ok(statisticsService.getMonthlyTrends(months));
    }
    
    @GetMapping("/specialities")
    public ResponseEntity<Map<String, Long>> getSpecialityDistribution() {
        return ResponseEntity.ok(statisticsService.getSpecialityDistribution());
    }
}

// PrescriptionController
@RestController
@RequestMapping("/api/prescriptions")
@CrossOrigin(origins = "${cors.allowed-origins}")
public class PrescriptionController {
    @Autowired
    private PrescriptionService prescriptionService;
    
    @GetMapping("/patient/{patientId}")
    public ResponseEntity<List<PrescriptionDTO>> getPatientPrescriptions(@PathVariable Long patientId) {
        return ResponseEntity.ok(prescriptionService.getActivePrescriptionsForPatient(patientId));
    }
    
    @PostMapping
    public ResponseEntity<PrescriptionDTO> createPrescription(@RequestBody CreatePrescriptionRequest request) {
        return ResponseEntity.status(201).body(prescriptionService.createPrescription(request));
    }
}
```

### 1.6 Update pom.xml with Modern Dependencies

```xml
<properties>
    <java.version>21</java.version>
    <spring-boot.version>4.0.5</spring-boot.version>
</properties>

<dependencies>
    <!-- Core Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Data Layer -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.33</version>
    </dependency>
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
    </dependency>
    
    <!-- Security & Auth -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Utilities -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.5.5.Final</version>
    </dependency>
    
    <!-- API Documentation -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.0.0</version>
    </dependency>
    
    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    
    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 1.7 Update Application Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/gestioncabinet?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: root
    password: ""
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect
  
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB

cors:
  allowed-origins: "http://localhost:3000,http://localhost:8080"
  allowed-methods: "GET,POST,PUT,DELETE,OPTIONS"
  allowed-headers: "*"
  allow-credentials: true

springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    enabled: true

server:
  port: 8080
```

---

## Phase 2: Enhanced Frontend Setup (Week 3-4)

### 2.1 React Project with Cabinet_Medical-Inspired UI

```bash
npx create-react-app gestion-cabinet-frontend --template typescript
cd gestion-cabinet-frontend

# Install dependencies
npm install \
  axios \
  react-router-dom \
  zustand \
  react-query \
  recharts \
  date-fns \
  react-calendar \
  react-modal \
  react-toastify \
  @hookform/resolvers \
  react-hook-form \
  zod

npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### 2.2 TypeScript Types (Enhanced)

```typescript
// types/index.ts

export enum AccountType {
  DOCTOR = 'DOCTOR',
  PATIENT = 'PATIENT',
  ADMIN = 'ADMIN',
  RECEPTIONIST = 'RECEPTIONIST'
}

export enum AppointmentStatus {
  SCHEDULED = 'SCHEDULED',
  IN_PROGRESS = 'IN_PROGRESS',
  COMPLETED = 'COMPLETED',
  CANCELLED = 'CANCELLED',
  NO_SHOW = 'NO_SHOW'
}

export interface User {
  id_user: number;
  cin: string;
  firstName: string;
  lastName: string;
  phone: string;
  email: string;
  account_type: AccountType;
  created_at: string;
}

export interface Patient extends User {
  id_patient: number;
  birthDate: string;
  sex: string;
  blood_type?: string;
  allergies?: string;
  medical_conditions?: string;
  insurance_number?: string;
  emergency_contact?: string;
  medicalFile?: MedicalFile;
}

export interface Doctor extends User {
  id_doctor: number;
  specialty: string;
  license_number: string;
  office_phone?: string;
  office_address?: string;
  years_of_experience?: number;
  is_available: boolean;
}

export interface Appointment {
  id_appointment: number;
  patient: Patient;
  doctor: Doctor;
  dateofAppointment: string;
  typeofIllness: string;
  description: string;
  status: AppointmentStatus;
  notification: boolean;
  notes?: string;
  created_at: string;
}

export interface Prescription {
  id_prescription: number;
  consultation: Consultation;
  patient: Patient;
  doctor: Doctor;
  medication_name: string;
  dosage: string;
  frequency: string;
  duration_days: number;
  instructions?: string;
  date_prescribed: string;
  is_active: boolean;
}

export interface Consultation {
  id_consultation: number;
  appointment: Appointment;
  patient: Patient;
  doctor: Doctor;
  date: string;
  description: string;
  diagnosis: string;
  treatment_plan: string;
  follow_up_date?: string;
  prescriptions: Prescription[];
  created_at: string;
}

export interface MedicalFile {
  id_medical_file: number;
  patient: Patient;
  total_consultations: number;
  last_consultation_date?: string;
  blood_type?: string;
  chronic_diseases?: string;
  surgeries?: string;
  vaccinations?: string;
}

export interface DashboardStats {
  totalPatients: number;
  totalDoctors: number;
  totalConsultations: number;
  consultationsThisMonth: number;
  specialityDistribution: { [key: string]: number };
}

export interface ConsultationTrend {
  month: string;
  consultations: number;
}
```

### 2.3 Cabinet_Medical-Inspired UI Components

```typescript
// components/common/AuthGuard.tsx
import { ReactNode } from 'react';
import { Navigate } from 'react-router-dom';
import { useAuthStore } from '../store/authStore';

interface AuthGuardProps {
  children: ReactNode;
  requiredRole?: AccountType[];
}

export const AuthGuard: React.FC<AuthGuardProps> = ({ children, requiredRole }) => {
  const { user } = useAuthStore();
  
  if (!user) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && !requiredRole.includes(user.account_type)) {
    return <Navigate to="/unauthorized" />;
  }
  
  return <>{children}</>;
};

// components/common/DoctorDashboard.tsx
import React, { useEffect } from 'react';
import { useAppointmentStore } from '../../store/appointmentStore';

export const DoctorDashboard: React.FC = () => {
  const { appointments, loading, fetchAppointments } = useAppointmentStore();
  
  useEffect(() => {
    fetchAppointments();
  }, []);
  
  if (loading) return <LoadingSpinner />;
  
  const upcomingCount = appointments.filter(a => a.status === 'SCHEDULED').length;
  const completedCount = appointments.filter(a => a.status === 'COMPLETED').length;
  
  return (
    <div className="dashboard-container">
      <div className="stats-cards">
        <StatCard title="Rendez-vous à venir" value={upcomingCount} icon="calendar" />
        <StatCard title="Consultations terminées" value={completedCount} icon="check" />
        <StatCard title="Patients" value={appointments.length} icon="users" />
      </div>
      
      <div className="recent-appointments">
        <h3>Rendez-vous récents</h3>
        <AppointmentList appointments={appointments.slice(0, 5)} />
      </div>
    </div>
  );
};

// components/appointments/AppointmentScheduler.tsx
import React, { useState } from 'react';
import Calendar from 'react-calendar';
import { useAppointmentStore } from '../../store/appointmentStore';
import 'react-calendar/dist/Calendar.css';

export const AppointmentScheduler: React.FC = () => {
  const [selectedDate, setSelectedDate] = useState<Date>(new Date());
  const [selectedDoctor, setSelectedDoctor] = useState<number | null>(null);
  const { doctors, createAppointment } = useAppointmentStore();
  
  const handleSchedule = async () => {
    // Check for conflicts
    const hasConflict = await checkAppointmentConflict(selectedDoctor, selectedDate);
    if (hasConflict) {
      alert('Cette plage horaire n\'est pas disponible');
      return;
    }
    
    await createAppointment({
      doctorId: selectedDoctor!,
      appointmentDate: selectedDate.toISOString(),
    });
  };
  
  return (
    <div className="appointment-scheduler">
      <h2>Prendre un rendez-vous</h2>
      
      <div className="scheduler-grid">
        <div>
          <h3>Sélectionner une date</h3>
          <Calendar onChange={setSelectedDate} value={selectedDate} />
        </div>
        
        <div>
          <h3>Sélectionner un médecin</h3>
          <DoctorSelect doctors={doctors} selected={selectedDoctor} onChange={setSelectedDoctor} />
          
          <button onClick={handleSchedule} className="btn-primary">
            Confirmer le rendez-vous
          </button>
        </div>
      </div>
    </div>
  );
};

// components/medical/MedicalFileView.tsx
import React from 'react';
import { MedicalFile, Consultation } from '../../types';

interface MedicalFileViewProps {
  medicalFile: MedicalFile;
  consultations: Consultation[];
}

export const MedicalFileView: React.FC<MedicalFileViewProps> = ({ medicalFile, consultations }) => {
  return (
    <div className="medical-file-container">
      <div className="file-header">
        <h2>Dossier Médical</h2>
        <p className="last-update">
          Dernière mise à jour: {new Date(medicalFile.updated_at).toLocaleDateString('fr-FR')}
        </p>
      </div>
      
      <div className="file-info-grid">
        <InfoCard label="Groupe sanguin" value={medicalFile.blood_type || 'N/A'} />
        <InfoCard label="Total consultations" value={medicalFile.total_consultations} />
        <InfoCard label="Maladies chroniques" value={medicalFile.chronic_diseases || 'Aucune'} />
      </div>
      
      <div className="consultations-history">
        <h3>Historique des consultations</h3>
        <ConsultationTimeline consultations={consultations} />
      </div>
    </div>
  );
};

// components/statistics/ConsultationAnalytics.tsx
import React from 'react';
import { LineChart, Line, BarChart, Bar, PieChart, Pie, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';

interface ConsultationAnalyticsProps {
  trends: Array<{ month: string; consultations: number }>;
  specialities: { [key: string]: number };
  doctorStats: Array<{ name: string; consultations: number }>;
}

export const ConsultationAnalytics: React.FC<ConsultationAnalyticsProps> = ({
  trends,
  specialities,
  doctorStats
}) => {
  return (
    <div className="analytics-container">
      <h2>Statistiques</h2>
      
      <div className="charts-grid">
        <div className="chart-card">
          <h3>Tendances mensuelles</h3>
          <ResponsiveContainer width="100%" height={300}>
            <LineChart data={trends}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="month" />
              <YAxis />
              <Tooltip />
              <Line type="monotone" dataKey="consultations" stroke="#8884d8" />
            </LineChart>
          </ResponsiveContainer>
        </div>
        
        <div className="chart-card">
          <h3>Distribution par spécialité</h3>
          <ResponsiveContainer width="100%" height={300}>
            <PieChart>
              <Pie data={Object.entries(specialities).map(([name, value]) => ({ name, value }))} 
                    dataKey="value" label />
              <Tooltip />
            </PieChart>
          </ResponsiveContainer>
        </div>
        
        <div className="chart-card">
          <h3>Performance des médecins</h3>
          <ResponsiveContainer width="100%" height={300}>
            <BarChart data={doctorStats}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="name" />
              <YAxis />
              <Tooltip />
              <Bar dataKey="consultations" fill="#82ca9d" />
            </BarChart>
          </ResponsiveContainer>
        </div>
      </div>
    </div>
  );
};
```

---

## Phase 3: Features Implementation (Week 4-6)

### 3.1 Appointment Booking with Conflict Detection
- Real-time availability calendar
- Automatic conflict detection (Cabinet_Medical inspired)
- Appointment reminders/notifications
- Cancellation with rescheduling

### 3.2 Prescription Management
- Create/edit prescriptions during consultation
- Prescription timeline for patients
- Medication interaction warnings (future enhancement)
- Prescription validity tracking

### 3.3 Medical File System
- Comprehensive patient medical history
- Timeline view of all consultations
- Diagnosis records with follow-ups
- Surgical history & vaccinations

### 3.4 Doctor Management & Performance Tracking
- Doctor profiles with specialities
- Availability schedules
- Performance metrics (consultations completed, patient reviews)
- Schedule management

### 3.5 Advanced Statistics Dashboard
- **Trends**: Monthly consultation trends
- **Speciality Distribution**: Pie chart of consultations by speciality
- **Doctor Performance**: Bar chart of doctor workload
- **Patient Demographics**: Age, gender, registration trends
- **Peak Hours**: Heatmap of busiest consultation times

---

## Phase 4: Authentication & Security (Week 6-7)

### 4.1 JWT Authentication
```java
// JwtTokenProvider
@Component
public class JwtTokenProvider {
    @Value("${jwt.secret:your-secret-key}")
    private String secret;
    
    @Value("${jwt.expiration:86400000}")
    private long expiration;
    
    public String generateToken(User user) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("accountType", user.getAccount_type());
        
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(user.getEmail())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secret).parseClaimsJws(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }
}

// JwtAuthenticationFilter
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        String token = getTokenFromRequest(request);
        
        if (token != null && tokenProvider.validateToken(token)) {
            // Set authentication in context
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getTokenFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### 4.2 Role-Based Access Control
```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .cors()
            .and()
            .authorizeRequests()
            .antMatchers("/api/auth/**").permitAll()
            .antMatchers("/api/doctors/**").permitAll()
            .antMatchers("/api/patients/**").hasAnyRole("DOCTOR", "ADMIN")
            .antMatchers("/api/appointments/**").hasAnyRole("DOCTOR", "PATIENT", "ADMIN")
            .antMatchers("/api/statistics/**").hasAnyRole("DOCTOR", "ADMIN")
            .anyRequest().authenticated()
            .and()
            .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

---

## Phase 5: Deployment & DevOps (Week 7)

### 5.1 Docker Setup

```dockerfile
# Backend Dockerfile
FROM openjdk:21-slim AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

COPY . .
RUN mvn clean package -DskipTests

FROM openjdk:21-slim
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]

# Frontend Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 5.2 Docker Compose

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: gestioncabinet
      MYSQL_ALLOW_EMPTY_PASSWORD: "no"
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10

  backend:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/gestioncabinet
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
    depends_on:
      mysql:
        condition: service_healthy
    volumes:
      - ./logs:/app/logs

  frontend:
    build:
      context: ./gestion-cabinet-frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    environment:
      REACT_APP_API_URL: http://backend:8080/api
    depends_on:
      - backend

volumes:
  mysql_data:
```

### 5.3 GitHub Actions CI/CD

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: gestioncabinet_test
          MYSQL_ROOT_PASSWORD: root
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 21
      uses: actions/setup-java@v3
      with:
        java-version: '21'
        distribution: 'temurin'
    
    - name: Run backend tests
      run: mvn clean test
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Run frontend tests
      run: |
        cd gestion-cabinet-frontend
        npm ci
        npm run test

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker images
      run: docker-compose build
    
    - name: Push to Docker Hub
      if: github.ref == 'refs/heads/main'
      run: |
        docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
        docker push ${{ secrets.DOCKER_REGISTRY }}/gestion-cabinet-backend:latest
        docker push ${{ secrets.DOCKER_REGISTRY }}/gestion-cabinet-frontend:latest
```

---

## Implementation Timeline

| Phase | Duration | Key Deliverables |
|-------|----------|-------------------|
| 1 | Week 1-3 | JPA entities, Repositories, Services, REST APIs |
| 2 | Week 3-4 | React setup, TypeScript types, State management |
| 3 | Week 4-6 | All features (Appointments, Prescriptions, Medical Files, Stats) |
| 4 | Week 6-7 | JWT Auth, Role-based access, Validation |
| 5 | Week 7 | Docker, Docker Compose, CI/CD |

**Total: 7 weeks (can be parallelized)**

---

## Key Improvements Over Cabinet_Medical

| Feature | Cabinet_Medical | GestionCabinet 2.0 |
|---------|-----------------|-------------------|
| Architecture | Servlet + JSP | Spring Boot + React SPA |
| Database | Manual JDBC + DAO | JPA/Hibernate ORM |
| Frontend | JSP + Bootstrap | React + Tailwind CSS + TypeScript |
| Real-time Features | Not supported | WebSocket support (future) |
| Mobile Support | Limited | Fully responsive |
| Testing | Manual testing | Automated CI/CD |
| Scalability | Limited | Microservices-ready |
| API Documentation | Not available | Swagger/OpenAPI |
| Authentication | Session-based | JWT + Spring Security |
| Performance Monitoring | Not included | Built-in metrics |

---

## Success Criteria

- ✅ 100% REST API coverage
- ✅ <2s page load time (frontend)
- ✅ 95%+ API test coverage
- ��� 0 critical security vulnerabilities
- ✅ Support for 1000+ concurrent users
- ✅ Mobile-responsive (all screen sizes)
- ✅ Automated deployment pipeline
- ✅ <30s appointment conflict check
- ✅ Real-time appointment notifications

---

## Technology Stack Comparison

```
Cabinet_Medical (JAVA EE)          GestionCabinet 2.0 (Modern)
├── Frontend                        ├── Frontend
│   ├── JSP                        │   ├── React 18
│   ├── HTML5                      │   ├── TypeScript
│   ├── Bootstrap                  │   ├── Tailwind CSS
│   └── JavaScript                 │   └── React Router v6
├── Backend                        ├── Backend
│   ├── Servlet/JSP               │   ├── Spring Boot 4.0.5
│   ├── DAO Pattern               │   ├── Spring Data JPA
│   └── Manual JDBC               │   ├── Hibernate 6
├── Database                       ├── Database
│   └── MySQL                      │   └── MySQL 8
└── Deployment                     └── Deployment
    └── Tomcat                         ├── Docker
                                       ├── Docker Compose
                                       └── GitHub Actions CI/CD
```

---

## Resources & Documentation

- **Appointment Conflict Detection**: Inspired by Cabinet_Medical's `AppointmentDAO.getAppointmentByDate()`
- **Notification System**: Adapted from Cabinet_Medical's notification flag & update logic
- **DAO to JPA Migration**: Cabinet_Medical's DAO layer modernized with Spring Data JPA
- **Frontend Layout**: Cabinet_Medical's JSP layout adapted to React components

