# ExamSeatDistributionSystem
[ExamSeatingSystem.java](https://github.com/user-attachments/files/23562250/ExamSeatingSystem.java)
import javax.swing.*;
import javax.swing.border.*;
import javax.swing.table.*;
import java.awt.*;
import java.awt.event.*;
import java.sql.*;
import java.util.*;

// Main Application Class
public class ExamSeatingSystem {
    public static void main(String[] args) {
        // Initialize database on startup
        DatabaseManager.initializeDatabase();
        
        SwingUtilities.invokeLater(() -> {
            LoginFrame frame = new LoginFrame();
            frame.setVisible(true);
            frame.setAlwaysOnTop(false);
            frame.toFront();
            frame.requestFocus();
        });
    }
}

// Database Connection Manager with H2
class DatabaseManager {
    private static final String DB_URL = "jdbc:h2:./examdb;AUTO_SERVER=TRUE";
    private static final String DB_USER = "sa";
    private static final String DB_PASSWORD = "";
    
    static {
        try {
            // Load H2 database driver
            Class.forName("org.h2.Driver");
        } catch (ClassNotFoundException e) {
            System.err.println("H2 Driver not found. Please ensure h2.jar is in the classpath.");
            e.printStackTrace();
        }
    }
    
    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
    }
    
    public static void initializeDatabase() {
        try (Connection conn = getConnection()) {
            Statement stmt = conn.createStatement();
            
            // Create users table
            stmt.execute(
                "CREATE TABLE IF NOT EXISTS users (" +
                "user_id INT AUTO_INCREMENT PRIMARY KEY," +
                "username VARCHAR(50) UNIQUE NOT NULL," +
                "password VARCHAR(100) NOT NULL," +
                "full_name VARCHAR(100) NOT NULL," +
                "email VARCHAR(100) UNIQUE NOT NULL," +
                "role VARCHAR(20) NOT NULL," +
                "created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP" +
                ")"
            );
            
            // Create exams table
            stmt.execute(
                "CREATE TABLE IF NOT EXISTS exams (" +
                "exam_id INT AUTO_INCREMENT PRIMARY KEY," +
                "exam_name VARCHAR(200) NOT NULL," +
                "exam_date DATE NOT NULL," +
                "exam_time TIME NOT NULL," +
                "duration_minutes INT NOT NULL," +
                "total_seats INT NOT NULL," +
                "created_by INT," +
                "created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP," +
                "FOREIGN KEY (created_by) REFERENCES users(user_id) ON DELETE SET NULL" +
                ")"
            );
            
            // Create rooms table
            stmt.execute(
                "CREATE TABLE IF NOT EXISTS rooms (" +
                "room_id INT AUTO_INCREMENT PRIMARY KEY," +
                "room_name VARCHAR(100) NOT NULL," +
                "capacity INT NOT NULL," +
                "building VARCHAR(100)," +
                "created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP" +
                ")"
            );
            
            // Create seating_arrangements table
            stmt.execute(
                "CREATE TABLE IF NOT EXISTS seating_arrangements (" +
                "arrangement_id INT AUTO_INCREMENT PRIMARY KEY," +
                "exam_id INT NOT NULL," +
                "student_id INT NOT NULL," +
                "room_id INT NOT NULL," +
                "seat_number VARCHAR(20) NOT NULL," +
                "created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP," +
                "FOREIGN KEY (exam_id) REFERENCES exams(exam_id) ON DELETE CASCADE," +
                "FOREIGN KEY (student_id) REFERENCES users(user_id) ON DELETE CASCADE," +
                "FOREIGN KEY (room_id) REFERENCES rooms(room_id) ON DELETE CASCADE" +
                ")"
            );
            
            // Insert default admin user if not exists
            PreparedStatement checkAdmin = conn.prepareStatement(
                "SELECT COUNT(*) FROM users WHERE username = 'admin'"
            );
            ResultSet rs = checkAdmin.executeQuery();
            rs.next();
            if (rs.getInt(1) == 0) {
                PreparedStatement insertAdmin = conn.prepareStatement(
                    "INSERT INTO users (username, password, full_name, email, role) " +
                    "VALUES ('admin', 'admin123', 'System Administrator', 'admin@example.com', 'admin')"
                );
                insertAdmin.executeUpdate();
            }
            
            // Insert sample rooms if not exists
            PreparedStatement checkRooms = conn.prepareStatement("SELECT COUNT(*) FROM rooms");
            rs = checkRooms.executeQuery();
            rs.next();
            if (rs.getInt(1) == 0) {
                stmt.execute("INSERT INTO rooms (room_name, capacity, building) VALUES ('Room 101', 30, 'Main Building')");
                stmt.execute("INSERT INTO rooms (room_name, capacity, building) VALUES ('Room 102', 40, 'Main Building')");
                stmt.execute("INSERT INTO rooms (room_name, capacity, building) VALUES ('Room 201', 35, 'Science Block')");
                stmt.execute("INSERT INTO rooms (room_name, capacity, building) VALUES ('Room 202', 50, 'Science Block')");
            }
            
            System.out.println("Database initialized successfully!");
            
        } catch (SQLException e) {
            System.err.println("Database initialization error: " + e.getMessage());
            e.printStackTrace();
        }
    }
}

// User Model
class User {
    private int userId;
    private String username;
    private String fullName;
    private String email;
    private String role;
    
    public User(int userId, String username, String fullName, String email, String role) {
        this.userId = userId;
        this.username = username;
        this.fullName = fullName;
        this.email = email;
        this.role = role;
    }
    
    public int getUserId() { return userId; }
    public String getUsername() { return username; }
    public String getFullName() { return fullName; }
    public String getEmail() { return email; }
    public String getRole() { return role; }
}

// Validation Utility
class Validator {
    public static boolean isValidName(String name) {
        return name != null && name.matches("^[a-zA-Z ]+$") && name.length() >= 2;
    }
    
    public static boolean isValidUsername(String username) {
        return username != null && username.matches("^[a-zA-Z0-9_]+$") && username.length() >= 4;
    }
    
    public static boolean isValidPassword(String password) {
        return password != null && password.matches("^[a-zA-Z0-9@#$%&*]+$") && password.length() >= 6;
    }
    
    public static boolean isValidEmail(String email) {
        return email != null && email.matches("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
    }
}

// Login Frame
class LoginFrame extends JFrame {
    private JTextField usernameField;
    private JPasswordField passwordField;
    
    public LoginFrame() {
        setTitle("Exam Seating System - Login");
        setSize(450, 550);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLocationRelativeTo(null);
        setResizable(false);
        setAlwaysOnTop(true);
        
        JPanel mainPanel = new JPanel();
        mainPanel.setLayout(new BorderLayout());
        mainPanel.setBackground(new Color(240, 242, 245));
        
        // Header Panel
        JPanel headerPanel = new JPanel();
        headerPanel.setBackground(new Color(41, 128, 185));
        headerPanel.setPreferredSize(new Dimension(450, 100));
        JLabel titleLabel = new JLabel("Exam Seating System");
        titleLabel.setFont(new Font("Segoe UI", Font.BOLD, 28));
        titleLabel.setForeground(Color.WHITE);
        headerPanel.add(titleLabel);
        
        // Form Panel
        JPanel formPanel = new JPanel();
        formPanel.setLayout(new GridBagLayout());
        formPanel.setBackground(Color.WHITE);
        formPanel.setBorder(new EmptyBorder(40, 50, 40, 50));
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.insets = new Insets(10, 0, 10, 0);
        
        JLabel loginLabel = new JLabel("Login to Your Account");
        loginLabel.setFont(new Font("Segoe UI", Font.BOLD, 20));
        loginLabel.setForeground(new Color(52, 73, 94));
        gbc.gridx = 0;
        gbc.gridy = 0;
        gbc.gridwidth = 2;
        formPanel.add(loginLabel, gbc);
        
        gbc.gridwidth = 1;
        gbc.gridy++;
        JLabel userLabel = new JLabel("Username:");
        userLabel.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        formPanel.add(userLabel, gbc);
        
        gbc.gridy++;
        usernameField = new JTextField(20);
        usernameField.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        usernameField.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(189, 195, 199)),
            BorderFactory.createEmptyBorder(8, 10, 8, 10)));
        formPanel.add(usernameField, gbc);
        
        gbc.gridy++;
        JLabel passLabel = new JLabel("Password:");
        passLabel.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        formPanel.add(passLabel, gbc);
        
        gbc.gridy++;
        passwordField = new JPasswordField(20);
        passwordField.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        passwordField.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(189, 195, 199)),
            BorderFactory.createEmptyBorder(8, 10, 8, 10)));
        formPanel.add(passwordField, gbc);
        
        gbc.gridy++;
        gbc.insets = new Insets(20, 0, 10, 0);
        JButton loginBtn = new JButton("Login");
        loginBtn.setFont(new Font("Segoe UI", Font.BOLD, 14));
        loginBtn.setBackground(new Color(41, 128, 185));
        loginBtn.setForeground(Color.WHITE);
        loginBtn.setFocusPainted(false);
        loginBtn.setBorder(new EmptyBorder(10, 0, 10, 0));
        loginBtn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        loginBtn.addActionListener(e -> handleLogin());
        formPanel.add(loginBtn, gbc);
        
        gbc.gridy++;
        gbc.insets = new Insets(10, 0, 10, 0);
        JButton registerBtn = new JButton("Register New Account");
        registerBtn.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        registerBtn.setBackground(new Color(149, 165, 166));
        registerBtn.setForeground(Color.WHITE);
        registerBtn.setFocusPainted(false);
        registerBtn.setBorder(new EmptyBorder(8, 0, 8, 0));
        registerBtn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        registerBtn.addActionListener(e -> {
            new RegistrationFrame().setVisible(true);
            dispose();
        });
        formPanel.add(registerBtn, gbc);
        
        // Add default credentials hint
        gbc.gridy++;
        JLabel hintLabel = new JLabel("<html><center><small>Default: admin / admin123</small></center></html>");
        hintLabel.setFont(new Font("Segoe UI", Font.ITALIC, 11));
        hintLabel.setForeground(Color.GRAY);
        formPanel.add(hintLabel, gbc);
        
        mainPanel.add(headerPanel, BorderLayout.NORTH);
        mainPanel.add(formPanel, BorderLayout.CENTER);
        
        add(mainPanel);
        
        // Enter key support
        passwordField.addActionListener(e -> handleLogin());
    }
    
    private void handleLogin() {
        String username = usernameField.getText().trim();
        String password = new String(passwordField.getPassword());
        
        if (username.isEmpty() || password.isEmpty()) {
            JOptionPane.showMessageDialog(this, "Please fill all fields!", "Error", JOptionPane.ERROR_MESSAGE);
            return;
        }
        
        try (Connection conn = DatabaseManager.getConnection()) {
            String query = "SELECT * FROM users WHERE username = ? AND password = ?";
            PreparedStatement pst = conn.prepareStatement(query);
            pst.setString(1, username);
            pst.setString(2, password);
            
            ResultSet rs = pst.executeQuery();
            
            if (rs.next()) {
                User user = new User(
                    rs.getInt("user_id"),
                    rs.getString("username"),
                    rs.getString("full_name"),
                    rs.getString("email"),
                    rs.getString("role")
                );
                
                JOptionPane.showMessageDialog(this, "Login Successful!\nWelcome, " + user.getFullName(), 
                    "Success", JOptionPane.INFORMATION_MESSAGE);
                
                openDashboard(user);
                dispose();
            } else {
                JOptionPane.showMessageDialog(this, "Invalid username or password!", 
                    "Error", JOptionPane.ERROR_MESSAGE);
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Database error: " + e.getMessage(), 
                "Error", JOptionPane.ERROR_MESSAGE);
        }
    }
    
    private void openDashboard(User user) {
        switch (user.getRole()) {
            case "admin":
                new AdminDashboard(user).setVisible(true);
                break;
            case "faculty":
                new FacultyDashboard(user).setVisible(true);
                break;
            case "student":
                new StudentDashboard(user).setVisible(true);
                break;
        }
    }
}

// Registration Frame
class RegistrationFrame extends JFrame {
    private JTextField usernameField, fullNameField, emailField;
    private JPasswordField passwordField, confirmPasswordField;
    private JComboBox<String> roleCombo;
    
    public RegistrationFrame() {
        setTitle("Register New Account");
        setSize(500, 700);
        setDefaultCloseOperation(JFrame.DISPOSE_ON_CLOSE);
        setLocationRelativeTo(null);
        setResizable(false);
        
        JPanel mainPanel = new JPanel();
        mainPanel.setLayout(new BorderLayout());
        mainPanel.setBackground(new Color(240, 242, 245));
        
        JPanel headerPanel = new JPanel();
        headerPanel.setBackground(new Color(39, 174, 96));
        headerPanel.setPreferredSize(new Dimension(500, 80));
        JLabel titleLabel = new JLabel("Create New Account");
        titleLabel.setFont(new Font("Segoe UI", Font.BOLD, 24));
        titleLabel.setForeground(Color.WHITE);
        headerPanel.add(titleLabel);
        
        JPanel formPanel = new JPanel();
        formPanel.setLayout(new GridBagLayout());
        formPanel.setBackground(Color.WHITE);
        formPanel.setBorder(new EmptyBorder(30, 40, 30, 40));
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.insets = new Insets(8, 0, 8, 0);
        gbc.gridx = 0;
        
        gbc.gridy = 0;
        formPanel.add(createLabel("Full Name (letters only):"), gbc);
        gbc.gridy++;
        fullNameField = createTextField();
        formPanel.add(fullNameField, gbc);
        
        gbc.gridy++;
        formPanel.add(createLabel("Username (alphanumeric):"), gbc);
        gbc.gridy++;
        usernameField = createTextField();
        formPanel.add(usernameField, gbc);
        
        gbc.gridy++;
        formPanel.add(createLabel("Email:"), gbc);
        gbc.gridy++;
        emailField = createTextField();
        formPanel.add(emailField, gbc);
        
        gbc.gridy++;
        formPanel.add(createLabel("Password (min 6 characters):"), gbc);
        gbc.gridy++;
        passwordField = createPasswordField();
        formPanel.add(passwordField, gbc);
        
        gbc.gridy++;
        formPanel.add(createLabel("Confirm Password:"), gbc);
        gbc.gridy++;
        confirmPasswordField = createPasswordField();
        formPanel.add(confirmPasswordField, gbc);
        
        gbc.gridy++;
        formPanel.add(createLabel("Role:"), gbc);
        gbc.gridy++;
        roleCombo = new JComboBox<>(new String[]{"student", "faculty"});
        roleCombo.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        formPanel.add(roleCombo, gbc);
        
        gbc.gridy++;
        gbc.insets = new Insets(20, 0, 10, 0);
        JButton registerBtn = new JButton("Register");
        registerBtn.setFont(new Font("Segoe UI", Font.BOLD, 14));
        registerBtn.setBackground(new Color(39, 174, 96));
        registerBtn.setForeground(Color.WHITE);
        registerBtn.setFocusPainted(false);
        registerBtn.setBorder(new EmptyBorder(10, 0, 10, 0));
        registerBtn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        registerBtn.addActionListener(e -> handleRegistration());
        formPanel.add(registerBtn, gbc);
        
        gbc.gridy++;
        gbc.insets = new Insets(5, 0, 5, 0);
        JButton backBtn = new JButton("Back to Login");
        backBtn.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        backBtn.setBackground(new Color(149, 165, 166));
        backBtn.setForeground(Color.WHITE);
        backBtn.setFocusPainted(false);
        backBtn.setBorder(new EmptyBorder(8, 0, 8, 0));
        backBtn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        backBtn.addActionListener(e -> {
            new LoginFrame().setVisible(true);
            dispose();
        });
        formPanel.add(backBtn, gbc);
        
        mainPanel.add(headerPanel, BorderLayout.NORTH);
        mainPanel.add(formPanel, BorderLayout.CENTER);
        add(mainPanel);
    }
    
    private JLabel createLabel(String text) {
        JLabel label = new JLabel(text);
        label.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        return label;
    }
    
    private JTextField createTextField() {
        JTextField field = new JTextField(25);
        field.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        field.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(189, 195, 199)),
            BorderFactory.createEmptyBorder(6, 8, 6, 8)));
        return field;
    }
    
    private JPasswordField createPasswordField() {
        JPasswordField field = new JPasswordField(25);
        field.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        field.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(189, 195, 199)),
            BorderFactory.createEmptyBorder(6, 8, 6, 8)));
        return field;
    }
    
    private void handleRegistration() {
        String fullName = fullNameField.getText().trim();
        String username = usernameField.getText().trim();
        String email = emailField.getText().trim();
        String password = new String(passwordField.getPassword());
        String confirmPass = new String(confirmPasswordField.getPassword());
        String role = (String) roleCombo.getSelectedItem();
        
        // Validation
        if (fullName.isEmpty() || username.isEmpty() || email.isEmpty() || 
            password.isEmpty() || confirmPass.isEmpty()) {
            JOptionPane.showMessageDialog(this, "All fields are required!", 
                "Error", JOptionPane.ERROR_MESSAGE);
            return;
        }
        
        if (!Validator.isValidName(fullName)) {
            JOptionPane.showMessageDialog(this, "Full name must contain only letters!", 
                "Error", JOptionPane.ERROR_MESSAGE);
            return;
        }
        
        if (!Validator.isValidUsername(username)) {
            JOptionPane.showMessageDialog(this, "Username must be alphanumeric (min 4 characters)!", 
                "Error", JOptionPane.ERROR_MESSAGE);
            return;
        }
        
        if (!Validator.isValidEmail(email)) {
            JOptionPane.showMessageDialog(this, "Invalid email format!", 
                "Error", JOptionPane.ERROR_MESSAGE);
            return;
        }
        
        if (!Validator.isValidPassword(password)) {
            JOptionPane.showMessageDialog(this, "Password must be alphanumeric (min 6 characters)!", 
                "Error", JOptionPane.ERROR_MESSAGE);
            return;
        }
        
        if (!password.equals(confirmPass)) {
            JOptionPane.showMessageDialog(this, "Passwords do not match!", 
                "Error", JOptionPane.ERROR_MESSAGE);
            return;
        }
        
        try (Connection conn = DatabaseManager.getConnection()) {
            String query = "INSERT INTO users (username, password, full_name, email, role) VALUES (?, ?, ?, ?, ?)";
            PreparedStatement pst = conn.prepareStatement(query);
            pst.setString(1, username);
            pst.setString(2, password);
            pst.setString(3, fullName);
            pst.setString(4, email);
            pst.setString(5, role);
            
            pst.executeUpdate();
            
            JOptionPane.showMessageDialog(this, "Registration Successful!\nYou can now login.", 
                "Success", JOptionPane.INFORMATION_MESSAGE);
            new LoginFrame().setVisible(true);
            dispose();
            
        } catch (SQLException e) {
            if (e.getMessage().contains("Unique")) {
                JOptionPane.showMessageDialog(this, "Username or email already exists!", 
                    "Error", JOptionPane.ERROR_MESSAGE);
            } else {
                JOptionPane.showMessageDialog(this, "Database error: " + e.getMessage(), 
                    "Error", JOptionPane.ERROR_MESSAGE);
            }
        }
    }
}

// Admin Dashboard (Continued in next part due to length...)
class AdminDashboard extends JFrame {
    private User user;
    private JPanel contentPanel;
    
    public AdminDashboard(User user) {
        this.user = user;
        setTitle("Admin Dashboard - " + user.getFullName());
        setSize(1000, 650);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLocationRelativeTo(null);
        
        JPanel mainPanel = new JPanel(new BorderLayout());
        
        // Sidebar
        JPanel sidebar = new JPanel();
        sidebar.setLayout(new BoxLayout(sidebar, BoxLayout.Y_AXIS));
        sidebar.setBackground(new Color(44, 62, 80));
        sidebar.setPreferredSize(new Dimension(200, 650));
        
        JLabel welcomeLabel = new JLabel("Admin Panel");
        welcomeLabel.setFont(new Font("Segoe UI", Font.BOLD, 18));
        welcomeLabel.setForeground(Color.WHITE);
        welcomeLabel.setAlignmentX(Component.CENTER_ALIGNMENT);
        welcomeLabel.setBorder(new EmptyBorder(20, 0, 20, 0));
        sidebar.add(welcomeLabel);
        
        sidebar.add(createMenuButton("Create Exam", e -> showCreateExam()));
        sidebar.add(createMenuButton("View Exams", e -> showViewExams()));
        sidebar.add(createMenuButton("Manage Rooms", e -> showManageRooms()));
        sidebar.add(createMenuButton("Assign Seating", e -> showAssignSeating()));
        sidebar.add(createMenuButton("View Users", e -> showViewUsers()));
        sidebar.add(Box.createVerticalGlue());
        sidebar.add(createMenuButton("Logout", e -> logout()));
        
        // Content Panel
        contentPanel = new JPanel(new CardLayout());
        contentPanel.setBackground(Color.WHITE);
        showDashboardHome();
        
        mainPanel.add(sidebar, BorderLayout.WEST);
        mainPanel.add(contentPanel, BorderLayout.CENTER);
        add(mainPanel);
    }
    
    private JButton createMenuButton(String text, ActionListener listener) {
        JButton btn = new JButton(text);
        btn.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        btn.setBackground(new Color(44, 62, 80));
        btn.setForeground(Color.WHITE);
        btn.setFocusPainted(false);
        btn.setBorderPainted(false);
        btn.setMaximumSize(new Dimension(200, 45));
        btn.setAlignmentX(Component.CENTER_ALIGNMENT);
        btn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        btn.addActionListener(listener);
        btn.addMouseListener(new MouseAdapter() {
            public void mouseEntered(MouseEvent e) {
                btn.setBackground(new Color(52, 73, 94));
            }
            public void mouseExited(MouseEvent e) {
                btn.setBackground(new Color(44, 62, 80));
            }
        });
        return btn;
    }
    
    private void showDashboardHome() {
        contentPanel.removeAll();
        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBackground(Color.WHITE);
        
        JLabel label = new JLabel("<html><center>Welcome to Admin Dashboard<br>" + 
            user.getFullName() + "<br><br>Select an option from the menu</center></html>");
        label.setFont(new Font("Segoe UI", Font.PLAIN, 20));
        label.setForeground(new Color(52, 73, 94));
        panel.add(label);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showCreateExam() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("Create New Exam");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(41, 128, 185));
        
        JPanel formPanel = new JPanel(new GridBagLayout());
        formPanel.setBackground(Color.WHITE);
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.insets = new Insets(10, 10, 10, 10);
        gbc.gridx = 0;
        
        gbc.gridy = 0;
        formPanel.add(new JLabel("Exam Name:"), gbc);
        gbc.gridx = 1;
        JTextField nameField = new JTextField(25);
        formPanel.add(nameField, gbc);
        
        gbc.gridx = 0;
        gbc.gridy++;
        formPanel.add(new JLabel("Date (YYYY-MM-DD):"), gbc);
        gbc.gridx = 1;
        JTextField dateField = new JTextField(25);
        formPanel.add(dateField, gbc);
        
        gbc.gridx = 0;
        gbc.gridy++;
        formPanel.add(new JLabel("Time (HH:MM):"), gbc);
        gbc.gridx = 1;
        JTextField timeField = new JTextField(25);
        formPanel.add(timeField, gbc);
        
        gbc.gridx = 0;
        gbc.gridy++;
        formPanel.add(new JLabel("Duration (minutes):"), gbc);
        gbc.gridx = 1;
        JTextField durationField = new JTextField(25);
        formPanel.add(durationField, gbc);
        
        gbc.gridx = 0;
        gbc.gridy++;
        formPanel.add(new JLabel("Total Seats:"), gbc);
        gbc.gridx = 1;
        JTextField seatsField = new JTextField(25);
        formPanel.add(seatsField, gbc);
        
        gbc.gridx = 1;
        gbc.gridy++;
        JButton createBtn = new JButton("Create Exam");
        createBtn.setBackground(new Color(39, 174, 96));
        createBtn.setForeground(Color.WHITE);
        createBtn.setFocusPainted(false);
        createBtn.addActionListener(e -> {
            try (Connection conn = DatabaseManager.getConnection()) {
                String query = "INSERT INTO exams (exam_name, exam_date, exam_time, duration_minutes, total_seats, created_by) VALUES (?, ?, ?, ?, ?, ?)";
                PreparedStatement pst = conn.prepareStatement(query);
                pst.setString(1, nameField.getText().trim());
                pst.setString(2, dateField.getText().trim());
                pst.setString(3, timeField.getText().trim());
                pst.setInt(4, Integer.parseInt(durationField.getText().trim()));
                pst.setInt(5, Integer.parseInt(seatsField.getText().trim()));
                pst.setInt(6, user.getUserId());
                
                pst.executeUpdate();
                JOptionPane.showMessageDialog(this, "Exam created successfully!");
                showViewExams();
            } catch (Exception ex) {
                JOptionPane.showMessageDialog(this, "Error: " + ex.getMessage(), "Error", JOptionPane.ERROR_MESSAGE);
            }
        });
        formPanel.add(createBtn, gbc);
        
        panel.add(title, BorderLayout.NORTH);
        panel.add(formPanel, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showViewExams() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("All Exams");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(41, 128, 185));
        
        String[] columns = {"ID", "Exam Name", "Date", "Time", "Duration", "Total Seats"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(25);
        table.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM exams ORDER BY exam_date DESC");
            
            while (rs.next()) {
                model.addRow(new Object[]{
                    rs.getInt("exam_id"),
                    rs.getString("exam_name"),
                    rs.getDate("exam_date"),
                    rs.getTime("exam_time"),
                    rs.getInt("duration_minutes") + " min",
                    rs.getInt("total_seats")
                });
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error loading exams: " + e.getMessage());
        }
        
        JScrollPane scrollPane = new JScrollPane(table);
        panel.add(title, BorderLayout.NORTH);
        panel.add(scrollPane, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showManageRooms() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("Manage Rooms");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(41, 128, 185));
        
        String[] columns = {"Room ID", "Room Name", "Capacity", "Building"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(25);
        
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM rooms");
            
            while (rs.next()) {
                model.addRow(new Object[]{
                    rs.getInt("room_id"),
                    rs.getString("room_name"),
                    rs.getInt("capacity"),
                    rs.getString("building")
                });
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        
        JScrollPane scrollPane = new JScrollPane(table);
        panel.add(title, BorderLayout.NORTH);
        panel.add(scrollPane, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showAssignSeating() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("Assign Seating Automatically");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(41, 128, 185));
        
        JPanel formPanel = new JPanel(new FlowLayout(FlowLayout.LEFT, 10, 10));
        formPanel.setBackground(Color.WHITE);
        
        formPanel.add(new JLabel("Select Exam:"));
        JComboBox<String> examCombo = new JComboBox<>();
        
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT exam_id, exam_name FROM exams ORDER BY exam_date DESC");
            
            while (rs.next()) {
                examCombo.addItem(rs.getInt("exam_id") + " - " + rs.getString("exam_name"));
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        
        formPanel.add(examCombo);
        
        JButton assignBtn = new JButton("Auto Assign Seats");
        assignBtn.setBackground(new Color(39, 174, 96));
        assignBtn.setForeground(Color.WHITE);
        assignBtn.setFocusPainted(false);
        assignBtn.addActionListener(e -> {
            if (examCombo.getSelectedItem() != null) {
                String selected = examCombo.getSelectedItem().toString();
                int examId = Integer.parseInt(selected.split(" - ")[0]);
                autoAssignSeats(examId);
            }
        });
        formPanel.add(assignBtn);
        
        panel.add(title, BorderLayout.NORTH);
        panel.add(formPanel, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void autoAssignSeats(int examId) {
        try (Connection conn = DatabaseManager.getConnection()) {
            // Clear existing assignments
            PreparedStatement clearPst = conn.prepareStatement("DELETE FROM seating_arrangements WHERE exam_id = ?");
            clearPst.setInt(1, examId);
            clearPst.executeUpdate();
            
            // Get all students
            Statement stmt = conn.createStatement();
            ResultSet students = stmt.executeQuery("SELECT user_id FROM users WHERE role = 'student'");
            
            // Get all rooms
            ResultSet rooms = stmt.executeQuery("SELECT room_id, capacity FROM rooms");
            
            ArrayList<Integer> studentList = new ArrayList<>();
            while (students.next()) {
                studentList.add(students.getInt("user_id"));
            }
            
            // Shuffle for random assignment
            Collections.shuffle(studentList);
            
            ArrayList<RoomInfo> roomList = new ArrayList<>();
            while (rooms.next()) {
                roomList.add(new RoomInfo(rooms.getInt("room_id"), rooms.getInt("capacity")));
            }
            
            // Assign students to rooms
            int studentIndex = 0;
            PreparedStatement assignPst = conn.prepareStatement(
                "INSERT INTO seating_arrangements (exam_id, student_id, room_id, seat_number) VALUES (?, ?, ?, ?)"
            );
            
            for (RoomInfo room : roomList) {
                for (int seat = 1; seat <= room.capacity && studentIndex < studentList.size(); seat++) {
                    assignPst.setInt(1, examId);
                    assignPst.setInt(2, studentList.get(studentIndex));
                    assignPst.setInt(3, room.roomId);
                    assignPst.setString(4, "S" + seat);
                    assignPst.executeUpdate();
                    studentIndex++;
                }
            }
            
            JOptionPane.showMessageDialog(this, 
                "Successfully assigned " + studentIndex + " students to seats!", 
                "Success", JOptionPane.INFORMATION_MESSAGE);
            
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage(), "Error", JOptionPane.ERROR_MESSAGE);
        }
    }
    
    private void showViewUsers() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("All Users");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(41, 128, 185));
        
        String[] columns = {"ID", "Username", "Full Name", "Email", "Role"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(25);
        table.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM users ORDER BY role, full_name");
            
            while (rs.next()) {
                model.addRow(new Object[]{
                    rs.getInt("user_id"),
                    rs.getString("username"),
                    rs.getString("full_name"),
                    rs.getString("email"),
                    rs.getString("role").toUpperCase()
                });
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        
        JScrollPane scrollPane = new JScrollPane(table);
        panel.add(title, BorderLayout.NORTH);
        panel.add(scrollPane, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void logout() {
        int choice = JOptionPane.showConfirmDialog(this, "Are you sure you want to logout?", 
            "Logout", JOptionPane.YES_NO_OPTION);
        if (choice == JOptionPane.YES_OPTION) {
            new LoginFrame().setVisible(true);
            dispose();
        }
    }
}

// Room Info Helper Class
class RoomInfo {
    int roomId;
    int capacity;
    
    public RoomInfo(int roomId, int capacity) {
        this.roomId = roomId;
        this.capacity = capacity;
    }
}

// Faculty Dashboard
class FacultyDashboard extends JFrame {
    private User user;
    private JPanel contentPanel;
    
    public FacultyDashboard(User user) {
        this.user = user;
        setTitle("Faculty Dashboard - " + user.getFullName());
        setSize(1000, 650);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLocationRelativeTo(null);
        
        JPanel mainPanel = new JPanel(new BorderLayout());
        
        JPanel sidebar = new JPanel();
        sidebar.setLayout(new BoxLayout(sidebar, BoxLayout.Y_AXIS));
        sidebar.setBackground(new Color(52, 73, 94));
        sidebar.setPreferredSize(new Dimension(200, 650));
        
        JLabel welcomeLabel = new JLabel("Faculty Panel");
        welcomeLabel.setFont(new Font("Segoe UI", Font.BOLD, 18));
        welcomeLabel.setForeground(Color.WHITE);
        welcomeLabel.setAlignmentX(Component.CENTER_ALIGNMENT);
        welcomeLabel.setBorder(new EmptyBorder(20, 0, 20, 0));
        sidebar.add(welcomeLabel);
        
        sidebar.add(createMenuButton("View Exams", e -> showViewExams()));
        sidebar.add(createMenuButton("View Seating", e -> showViewSeating()));
        sidebar.add(createMenuButton("View Rooms", e -> showViewRooms()));
        sidebar.add(Box.createVerticalGlue());
        sidebar.add(createMenuButton("Logout", e -> logout()));
        
        contentPanel = new JPanel(new CardLayout());
        contentPanel.setBackground(Color.WHITE);
        showDashboardHome();
        
        mainPanel.add(sidebar, BorderLayout.WEST);
        mainPanel.add(contentPanel, BorderLayout.CENTER);
        add(mainPanel);
    }
    
    private JButton createMenuButton(String text, ActionListener listener) {
        JButton btn = new JButton(text);
        btn.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        btn.setBackground(new Color(52, 73, 94));
        btn.setForeground(Color.WHITE);
        btn.setFocusPainted(false);
        btn.setBorderPainted(false);
        btn.setMaximumSize(new Dimension(200, 45));
        btn.setAlignmentX(Component.CENTER_ALIGNMENT);
        btn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        btn.addActionListener(listener);
        btn.addMouseListener(new MouseAdapter() {
            public void mouseEntered(MouseEvent e) {
                btn.setBackground(new Color(71, 91, 112));
            }
            public void mouseExited(MouseEvent e) {
                btn.setBackground(new Color(52, 73, 94));
            }
        });
        return btn;
    }
    
    private void showDashboardHome() {
        contentPanel.removeAll();
        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBackground(Color.WHITE);
        
        JLabel label = new JLabel("<html><center>Welcome Faculty Member<br>" + 
            user.getFullName() + "<br><br>Select an option from the menu</center></html>");
        label.setFont(new Font("Segoe UI", Font.PLAIN, 20));
        label.setForeground(new Color(52, 73, 94));
        panel.add(label);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showViewExams() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("All Exams");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(52, 73, 94));
        
        String[] columns = {"ID", "Exam Name", "Date", "Time", "Duration", "Total Seats"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(25);
        table.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM exams ORDER BY exam_date DESC");
            
            while (rs.next()) {
                model.addRow(new Object[]{
                    rs.getInt("exam_id"),
                    rs.getString("exam_name"),
                    rs.getDate("exam_date"),
                    rs.getTime("exam_time"),
                    rs.getInt("duration_minutes") + " min",
                    rs.getInt("total_seats")
                });
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        
        JScrollPane scrollPane = new JScrollPane(table);
        panel.add(title, BorderLayout.NORTH);
        panel.add(scrollPane, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showViewSeating() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("Seating Arrangements");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(52, 73, 94));
        
        JPanel topPanel = new JPanel(new FlowLayout(FlowLayout.LEFT));
        topPanel.setBackground(Color.WHITE);
        topPanel.add(new JLabel("Select Exam: "));
        
        JComboBox<String> examCombo = new JComboBox<>();
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT exam_id, exam_name FROM exams ORDER BY exam_date DESC");
            
            while (rs.next()) {
                examCombo.addItem(rs.getInt("exam_id") + " - " + rs.getString("exam_name"));
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        topPanel.add(examCombo);
        
        String[] columns = {"Student Name", "Room", "Seat Number"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(25);
        table.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        
        JButton viewBtn = new JButton("View Seating");
        viewBtn.setBackground(new Color(52, 73, 94));
        viewBtn.setForeground(Color.WHITE);
        viewBtn.setFocusPainted(false);
        viewBtn.addActionListener(e -> {
            if (examCombo.getSelectedItem() != null) {
                model.setRowCount(0);
                String selected = examCombo.getSelectedItem().toString();
                int examId = Integer.parseInt(selected.split(" - ")[0]);
                
                try (Connection conn = DatabaseManager.getConnection()) {
                    String query = "SELECT u.full_name, r.room_name, sa.seat_number " +
                                   "FROM seating_arrangements sa " +
                                   "JOIN users u ON sa.student_id = u.user_id " +
                                   "JOIN rooms r ON sa.room_id = r.room_id " +
                                   "WHERE sa.exam_id = ? " +
                                   "ORDER BY r.room_name, sa.seat_number";
                    PreparedStatement pst = conn.prepareStatement(query);
                    pst.setInt(1, examId);
                    ResultSet rs = pst.executeQuery();
                    
                    while (rs.next()) {
                        model.addRow(new Object[]{
                            rs.getString("full_name"),
                            rs.getString("room_name"),
                            rs.getString("seat_number")
                        });
                    }
                } catch (SQLException ex) {
                    JOptionPane.showMessageDialog(this, "Error: " + ex.getMessage());
                }
            }
        });
        topPanel.add(viewBtn);
        
        JScrollPane scrollPane = new JScrollPane(table);
        
        JPanel mainContentPanel = new JPanel(new BorderLayout(10, 10));
        mainContentPanel.setBackground(Color.WHITE);
        mainContentPanel.add(title, BorderLayout.NORTH);
        mainContentPanel.add(topPanel, BorderLayout.CENTER);
        mainContentPanel.add(scrollPane, BorderLayout.SOUTH);
        
        panel.add(mainContentPanel);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showViewRooms() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("Available Rooms");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(52, 73, 94));
        
        String[] columns = {"Room ID", "Room Name", "Capacity", "Building"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(25);
        
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM rooms");
            
            while (rs.next()) {
                model.addRow(new Object[]{
                    rs.getInt("room_id"),
                    rs.getString("room_name"),
                    rs.getInt("capacity"),
                    rs.getString("building")
                });
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        
        JScrollPane scrollPane = new JScrollPane(table);
        panel.add(title, BorderLayout.NORTH);
        panel.add(scrollPane, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void logout() {
        int choice = JOptionPane.showConfirmDialog(this, "Are you sure you want to logout?", 
            "Logout", JOptionPane.YES_NO_OPTION);
        if (choice == JOptionPane.YES_OPTION) {
            new LoginFrame().setVisible(true);
            dispose();
        }
    }
}

// Student Dashboard
class StudentDashboard extends JFrame {
    private User user;
    private JPanel contentPanel;
    
    public StudentDashboard(User user) {
        this.user = user;
        setTitle("Student Dashboard - " + user.getFullName());
        setSize(900, 600);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLocationRelativeTo(null);
        
        JPanel mainPanel = new JPanel(new BorderLayout());
        
        JPanel sidebar = new JPanel();
        sidebar.setLayout(new BoxLayout(sidebar, BoxLayout.Y_AXIS));
        sidebar.setBackground(new Color(41, 128, 185));
        sidebar.setPreferredSize(new Dimension(200, 600));
        
        JLabel welcomeLabel = new JLabel("Student Panel");
        welcomeLabel.setFont(new Font("Segoe UI", Font.BOLD, 18));
        welcomeLabel.setForeground(Color.WHITE);
        welcomeLabel.setAlignmentX(Component.CENTER_ALIGNMENT);
        welcomeLabel.setBorder(new EmptyBorder(20, 0, 20, 0));
        sidebar.add(welcomeLabel);
        
        sidebar.add(createMenuButton("My Seating", e -> showMySeating()));
        sidebar.add(createMenuButton("All Exams", e -> showAllExams()));
        sidebar.add(Box.createVerticalGlue());
        sidebar.add(createMenuButton("Logout", e -> logout()));
        
        contentPanel = new JPanel(new CardLayout());
        contentPanel.setBackground(Color.WHITE);
        showDashboardHome();
        
        mainPanel.add(sidebar, BorderLayout.WEST);
        mainPanel.add(contentPanel, BorderLayout.CENTER);
        add(mainPanel);
    }
    
    private JButton createMenuButton(String text, ActionListener listener) {
        JButton btn = new JButton(text);
        btn.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        btn.setBackground(new Color(41, 128, 185));
        btn.setForeground(Color.WHITE);
        btn.setFocusPainted(false);
        btn.setBorderPainted(false);
        btn.setMaximumSize(new Dimension(200, 45));
        btn.setAlignmentX(Component.CENTER_ALIGNMENT);
        btn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        btn.addActionListener(listener);
        btn.addMouseListener(new MouseAdapter() {
            public void mouseEntered(MouseEvent e) {
                btn.setBackground(new Color(52, 152, 219));
            }
            public void mouseExited(MouseEvent e) {
                btn.setBackground(new Color(41, 128, 185));
            }
        });
        return btn;
    }
    
    private void showDashboardHome() {
        contentPanel.removeAll();
        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBackground(Color.WHITE);
        
        JLabel label = new JLabel("<html><center>Welcome Student<br>" + 
            user.getFullName() + "<br><br>Check your exam seating arrangements</center></html>");
        label.setFont(new Font("Segoe UI", Font.PLAIN, 20));
        label.setForeground(new Color(52, 73, 94));
        panel.add(label);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showMySeating() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("My Seating Arrangements");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(41, 128, 185));
        
        String[] columns = {"Exam Name", "Date", "Time", "Room", "Seat Number"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(30);
        table.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        
        try (Connection conn = DatabaseManager.getConnection()) {
            String query = "SELECT e.exam_name, e.exam_date, e.exam_time, r.room_name, sa.seat_number " +
                           "FROM seating_arrangements sa " +
                           "JOIN exams e ON sa.exam_id = e.exam_id " +
                           "JOIN rooms r ON sa.room_id = r.room_id " +
                           "WHERE sa.student_id = ? " +
                           "ORDER BY e.exam_date, e.exam_time";
            PreparedStatement pst = conn.prepareStatement(query);
            pst.setInt(1, user.getUserId());
            ResultSet rs = pst.executeQuery();
            
            while (rs.next()) {
                model.addRow(new Object[]{
                    rs.getString("exam_name"),
                    rs.getDate("exam_date"),
                    rs.getTime("exam_time"),
                    rs.getString("room_name"),
                    rs.getString("seat_number")
                });
            }
            
            if (model.getRowCount() == 0) {
                JLabel noData = new JLabel("No seating arrangements assigned yet.");
                noData.setFont(new Font("Segoe UI", Font.PLAIN, 16));
                noData.setForeground(Color.GRAY);
                JPanel noDataPanel = new JPanel(new GridBagLayout());
                noDataPanel.setBackground(Color.WHITE);
                noDataPanel.add(noData);
                panel.add(noDataPanel, BorderLayout.CENTER);
            } else {
                JScrollPane scrollPane = new JScrollPane(table);
                panel.add(scrollPane, BorderLayout.CENTER);
            }
            
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        
        panel.add(title, BorderLayout.NORTH);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void showAllExams() {
        contentPanel.removeAll();
        
        JPanel panel = new JPanel(new BorderLayout(10, 10));
        panel.setBackground(Color.WHITE);
        panel.setBorder(new EmptyBorder(20, 20, 20, 20));
        
        JLabel title = new JLabel("Upcoming Exams");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(41, 128, 185));
        
        String[] columns = {"Exam Name", "Date", "Time", "Duration"};
        DefaultTableModel model = new DefaultTableModel(columns, 0);
        JTable table = new JTable(model);
        table.setRowHeight(30);
        table.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        
        try (Connection conn = DatabaseManager.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM exams ORDER BY exam_date DESC");
            
            while (rs.next()) {
                model.addRow(new Object[]{
                    rs.getString("exam_name"),
                    rs.getDate("exam_date"),
                    rs.getTime("exam_time"),
                    rs.getInt("duration_minutes") + " min"
                });
            }
        } catch (SQLException e) {
            JOptionPane.showMessageDialog(this, "Error: " + e.getMessage());
        }
        
        JScrollPane scrollPane = new JScrollPane(table);
        panel.add(title, BorderLayout.NORTH);
        panel.add(scrollPane, BorderLayout.CENTER);
        
        contentPanel.add(panel);
        contentPanel.revalidate();
        contentPanel.repaint();
    }
    
    private void logout() {
        int choice = JOptionPane.showConfirmDialog(this, "Are you sure you want to logout?", 
            "Logout", JOptionPane.YES_NO_OPTION);
        if (choice == JOptionPane.YES_OPTION) {
            new LoginFrame().setVisible(true);
            dispose();
        }
    }
}
## 🛠 PROCESS (How the System Works)

1. **Start the Application**  
   The user runs the program and the Exam Seating System window opens.

2. **Enter Student Details**  
   The user enters:
   - Student Name
   - Roll Number
   - Class / Course
   - Exam Name

3. **Enter Seating Details**  
   The user fills:
   - Room Number
   - Bench Number / Seat Number
   - Row or Column (if required)

4. **Generate Seating Output**  
   After entering all details, the user clicks the **Generate** or **Submit** button.
   The system displays the seating information in a proper formatted output.

5. **Print / Save the Seating Info**  
   The user can:
   - Print the seating card  
   - Save the details  
   - Use the output for exam room planning

6. **Reset / New Entry**  
   The user can clear fields using **Reset** and add new student seating details.
