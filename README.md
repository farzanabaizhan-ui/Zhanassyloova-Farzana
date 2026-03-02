package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;
import org.springframework.stereotype.Service;
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import jakarta.validation.Valid;

import java.util.*;

import org.springframework.data.jpa.repository.JpaRepository;
import lombok.RequiredArgsConstructor;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}

@Entity
class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "Name cannot be empty") 
    private String name;

    @Email(message = "Invalid email")
    private String email;

    // One-to-One
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL)
    private Profile profile;

    // One-to-Many
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<OrderEntity> orders = new ArrayList<>();

    // getters & setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public Profile getProfile() { return profile; }
    public List<OrderEntity> getOrders() { return orders; }

    public void setId(Long id) { this.id = id; }
    public void setName(String name) { this.name = name; }
    public void setEmail(String email) { this.email = email; }
    public void setProfile(Profile profile) { this.profile = profile; }
    public void setOrders(List<OrderEntity> orders) { this.orders = orders; }
}

@Entity
class Profile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String address;

    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;

    public Long getId() { return id; }
    public String getAddress() { return address; }
    public User getUser() { return user; }

    public void setId(Long id) { this.id = id; }
    public void setAddress(String address) { this.address = address; }
    public void setUser(User user) { this.user = user; }
}

@Entity
@Table(name = "orders")
class OrderEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull(message = "Product cannot be null")  // validation 1
    private String productName;

    private Double price;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    public Long getId() { return id; }
    public String getProductName() { return productName; }
    public Double getPrice() { return price; }
    public User getUser() { return user; }

    public void setId(Long id) { this.id = id; }
    public void setProductName(String productName) { this.productName = productName; }
    public void setPrice(Double price) { this.price = price; }
    public void setUser(User user) { this.user = user; }
}

interface UserRepository extends JpaRepository<User, Long> {}
interface ProfileRepository extends JpaRepository<Profile, Long> {}
interface OrderRepository extends JpaRepository<OrderEntity, Long> {}

@Service
@RequiredArgsConstructor
class UserService {

    private final UserRepository userRepository;

    public User create(User user) {

        if (user.getName().length() < 3) {
            throw new IllegalArgumentException("Name must be at least 3 characters");
        }

        return userRepository.save(user);
    }

    public List<User> getAll() {
        return userRepository.findAll();
    }

    public User getOne(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("User not found"));
    }

    public User update(Long id, User newUser) {
        User user = getOne(id);
        user.setName(newUser.getName());
        user.setEmail(newUser.getEmail());
        return userRepository.save(user);
    }

    public void delete(Long id) {
        User user = getOne(id);
        userRepository.delete(user);
    }
}

@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
class UserController {

    private final UserService service;

    @PostMapping
    public User create(@Valid @RequestBody User user) {
        return service.create(user);
    }

    @GetMapping
    public List<User> getAll() {
        return service.getAll();
    }

    @GetMapping("/{id}")
    public User getOne(@PathVariable Long id) {
        return service.getOne(id);
    }

    @PutMapping("/{id}")
    public User update(@PathVariable Long id,
                       @RequestBody User user) {
        return service.update(id, user);
    }

    @DeleteMapping("/{id}")
    public String delete(@PathVariable Long id) {
        service.delete(id);
        return "User deleted successfully";
    }
}

class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
        return new ResponseEntity<>(ex.getMessage(), HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<String> handleValidation(MethodArgumentNotValidException ex) {
        return new ResponseEntity<>("Validation error: " +
                ex.getBindingResult().getFieldError().getDefaultMessage(),
                HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleManualValidation(IllegalArgumentException ex) {
        return new ResponseEntity<>(ex.getMessage(), HttpStatus.BAD_REQUEST);
    }
}
