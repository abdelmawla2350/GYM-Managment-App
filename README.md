# GYM-Managment-App

import java.awt.*;
import java.io.*;
import java.time.*;
import java.time.format.*;
import java.util.*;
import java.util.List;
import java.util.stream.*;
import javax.swing.*;
import javax.swing.table.*;

public class GymAppCombined {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(MainPage::new);
    }

    // ---------- Rounded Button Class ----------
    static class RoundedButton extends JButton {
        public RoundedButton(String label) {
            super(label);
            setFocusPainted(false);
            setContentAreaFilled(false);
            setFont(new Font("Arial", Font.BOLD, 16));
            setForeground(Color.BLACK);
        }

        protected void paintComponent(Graphics g) {
            Graphics2D g2 = (Graphics2D) g.create();
            g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
            g2.setColor(getModel().isArmed() ? new Color(70, 130, 180) : new Color(100, 149, 237));
            g2.fillRoundRect(0, 0, getWidth(), getHeight(), 30, 30);
            g2.dispose();
            super.paintComponent(g);
        }

        protected void paintBorder(Graphics g) {
            g.setColor(Color.DARK_GRAY);
            g.drawRoundRect(0, 0, getWidth() - 1, getHeight() - 1, 30, 30);
        }

        public Dimension getPreferredSize() {
            FontMetrics fm = getFontMetrics(getFont());
            int width = fm.stringWidth(getText()) + 40;
            int height = fm.getHeight() + 20;
            return new Dimension(Math.max(width, 140), Math.max(height, 50));
        }
    }

    // ---------- Member Data Class ----------
    static class Member {
        String id;
        String name;
        String gender;
        String email;
        String plan;
        int age;
        int phonenumber;
        LocalDate startdate;
        LocalDate enddate;

        public Member(String id, String name, int age, String gender, String email,
                     String plan, int phonenumber, LocalDate startdate, LocalDate enddate) {
            this.id = id;
            this.name = name;
            this.age = age;
            this.gender = gender;
            this.email = email;
            this.plan = plan;
            this.phonenumber = phonenumber;
            this.startdate = startdate;
            this.enddate = enddate;
        }

        public String toCSV() {
            return String.join(",",
                    id, name, String.valueOf(age), gender,
                    String.valueOf(phonenumber), email, plan,
                    startdate.toString(), enddate.toString());
        }

        public static Member fromCSV(String csv) {
            String[] parts = csv.split(",");
            return new Member(
                    parts[0], parts[1], Integer.parseInt(parts[2]),
                    parts[3], parts[5], parts[6],
                    Integer.parseInt(parts[4]),
                    LocalDate.parse(parts[7]),
                    LocalDate.parse(parts[8])
            );
        }
    }

    // ---------- Data Handling ----------
    static class MemberStorage {
        private static final String FILE = "members.txt";
        static List<Member> members = new ArrayList<>();

        static void loadMembers() {
            members.clear();
            try (BufferedReader br = new BufferedReader(new FileReader(FILE))) {
                String line;
                while ((line = br.readLine()) != null) {
                    if (!line.trim().isEmpty()) {
                        members.add(Member.fromCSV(line));
                    }
                }
            } catch (IOException e) {
                // File will be created on first save
            }
        }

        static void saveMembers() {
            try (PrintWriter pw = new PrintWriter(new FileWriter(FILE))) {
                members.forEach(m -> pw.println(m.toCSV()));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        static Member getByID(String id) {
            return members.stream()
                    .filter(m -> m.id.equals(id))
                    .findFirst()
                    .orElse(null);
        }

        static List<Member> getActiveMembers() {
            return members.stream()
                    .filter(m -> !m.enddate.isBefore(LocalDate.now()))
                    .collect(Collectors.toList());
        }

        static List<Member> getInactiveMembers() {
            return members.stream()
                    .filter(m -> m.enddate.isBefore(LocalDate.now()))
                    .collect(Collectors.toList());
        }

        public static boolean addMember(Member newMember) {
            if (getByID(newMember.id) != null) {
                return false; // Member already exists
            }
            members.add(newMember);
            saveMembers();
            return true;
        }

        public static boolean editMember(String id, String newName, String newEmail, String newPhone) {
            Member m = getByID(id);
            if (m != null) {
                m.name = newName;
                m.email = newEmail;
                m.phonenumber = Integer.parseInt(newPhone);
                saveMembers();
                return true;
            }
            return false;
        }

        public static boolean removeMember(String id) {
            Member m = getByID(id);
            if (m != null) {
                members.remove(m);
                saveMembers();
                return true;
            }
            return false;
        }
    }

    // ---------- Activity Logger ----------
    static class ActivityLogger {
        private static final String LOG_FILE = "activities.log";

        public static void logActivity(String activity) {
            String timestamp = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
            try (PrintWriter pw = new PrintWriter(new FileWriter(LOG_FILE, true))) {
                pw.println(timestamp + "|" + activity);
            } catch (IOException e) {
                JOptionPane.showMessageDialog(null, "Failed to log activity", "Error", JOptionPane.ERROR_MESSAGE);
            }
        }

        public static List<String[]> getRecentActivities(int count) {
            List<String[]> activities = new ArrayList<>();
            try (BufferedReader br = new BufferedReader(new FileReader(LOG_FILE))) {
                String line;
                while ((line = br.readLine()) != null && activities.size() < count) {
                    String[] parts = line.split("\\|", 2);
                    if (parts.length == 2) {
                        activities.add(new String[]{parts[0], parts[1]});
                    }
                }
            } catch (IOException e) {
                // No logging needed for missing file
            }
            return activities;
        }
    }

// ---------- Reports Generator ----------
static class ReportsGenerator {
    public static void generateMonthlyReport() {
        try {
            // Ensure fresh data is loaded
            MemberStorage.loadMembers();
            
            // Create main report panel
            JPanel reportPanel = new JPanel(new BorderLayout());
            reportPanel.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));

            // 1. Member Statistics Tab
            String[] statColumns = {"Metric", "Value"};
            Object[][] statData = {
                {"Total Members", MemberStorage.members.size()},
                {"Active Members", MemberStorage.getActiveMembers().size()},
                {"Inactive Members", MemberStorage.getInactiveMembers().size()},
                {"Most Popular Plan", getMostPopularPlan()},
                {"Avg Member Age", getAverageAge()}
            };
            JTable statsTable = createTable(statColumns, statData);

            // 2. Membership Plans Breakdown
            Map<String, Long> planCounts = MemberStorage.members.stream()
                .collect(Collectors.groupingBy(m -> m.plan, Collectors.counting()));
            
            String[] planColumns = {"Plan", "Subscribers", "Percentage"};
            Object[][] planData = planCounts.entrySet().stream()
                .map(e -> new Object[]{
                    e.getKey(), 
                    e.getValue(),
                    String.format("%.1f%%", (e.getValue() * 100.0 / MemberStorage.members.size()))
                })
                .toArray(Object[][]::new);
            JTable plansTable = createTable(planColumns, planData);

            // 3. Recent Activities Log
            String[] activityColumns = {"Date", "Time", "Activity"};
            List<String[]> rawActivities = ActivityLogger.getRecentActivities(20);
            Object[][] activityData = rawActivities.stream()
                .map(a -> {
                    String[] dt = a[0].split("T");
                    return new Object[]{
                        dt[0], // Date
                        dt[1].substring(0, 8), // Time
                        a[1] // Activity
                    };
                })
                .toArray(Object[][]::new);
            JTable activityTable = createTable(activityColumns, activityData);

            // Create tabbed interface
            JTabbedPane tabbedPane = new JTabbedPane();
            tabbedPane.addTab("Statistics", new JScrollPane(statsTable));
            tabbedPane.addTab("Membership Plans", new JScrollPane(plansTable));
            tabbedPane.addTab("Recent Activities", new JScrollPane(activityTable));
            
            reportPanel.add(tabbedPane, BorderLayout.CENTER);
            
            // Create and show report dialog
            JDialog reportDialog = new JDialog();
            reportDialog.setTitle("Monthly Gym Report - " + LocalDate.now().getMonth());
            reportDialog.setModal(true);
            reportDialog.setContentPane(reportPanel);
            reportDialog.pack();
            reportDialog.setSize(800, 600);
            reportDialog.setLocationRelativeTo(null);
            reportDialog.setVisible(true);

        } catch (Exception e) {
            JOptionPane.showMessageDialog(null, 
                "Error generating report: " + e.getMessage(), 
                "Report Error", 
                JOptionPane.ERROR_MESSAGE);
            e.printStackTrace();
        }
    }

    private static JTable createTable(String[] columns, Object[][] data) {
        DefaultTableModel model = new DefaultTableModel(data, columns) {
            @Override public boolean isCellEditable(int row, int column) {
                return false;
            }
        };
        JTable table = new JTable(model);
        table.setAutoCreateRowSorter(true);
        table.setFillsViewportHeight(true);
        return table;
    }

    private static String getMostPopularPlan() {
        return MemberStorage.members.stream()
            .collect(Collectors.groupingBy(m -> m.plan, Collectors.counting()))
            .entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElse("No data");
    }

    private static double getAverageAge() {
        return MemberStorage.members.stream()
            .mapToInt(m -> m.age)
            .average()
            .orElse(0.0);
    }
}

    // ---------- Member Login Page ----------
    static class LoginPage extends JFrame {
        private ImageIcon icon = new ImageIcon("weight.png");

        public LoginPage() {
            MemberStorage.loadMembers();
            setIconImage(icon.getImage());
            setTitle("Member Login");
            setSize(400, 300);
            setDefaultCloseOperation(EXIT_ON_CLOSE);
            setLayout(new BorderLayout());

            JPanel topPanel = new JPanel();
            topPanel.setBackground(new Color(169, 169, 169));
            JLabel title = new JLabel("Member Login", SwingConstants.CENTER);
            title.setFont(new Font("Arial", Font.BOLD, 20));
            title.setForeground(Color.WHITE);
            topPanel.add(title);
            add(topPanel, BorderLayout.NORTH);

            JPanel centerPanel = new JPanel(new GridLayout(3, 2, 10, 10));
            centerPanel.setBackground(new Color(220, 220, 220));
            centerPanel.setBorder(BorderFactory.createEmptyBorder(20, 20, 20, 20));

            JLabel idLabel = new JLabel("Member ID:");
            JTextField idField = new JTextField();
            JLabel pinLabel = new JLabel("PIN:");
            JPasswordField pinField = new JPasswordField();
            RoundedButton loginBtn = new RoundedButton("Login");
            RoundedButton backBtn = new RoundedButton("Back");

            centerPanel.add(idLabel);
            centerPanel.add(idField);
            centerPanel.add(pinLabel);
            centerPanel.add(pinField);
            centerPanel.add(loginBtn);
            centerPanel.add(backBtn);

            loginBtn.addActionListener(e -> {
                String id = idField.getText().trim();
                Member member = MemberStorage.getByID(id);
                
                if (member != null) {
                    ActivityLogger.logActivity("Member login: " + member.name);
                    dispose(); // Close current window
                    new MemberDashboard(member);
                }
            });
            
            backBtn.addActionListener(e -> {
                dispose(); // Close current window
                new MainPage();
            });

            add(centerPanel, BorderLayout.CENTER);
            setLocationRelativeTo(null);
            setVisible(true);
        }
    }

    // ---------- client Registration ----------
    static class RegisterNewMember extends JFrame {
        private ImageIcon icon = new ImageIcon("weight.png");

        public RegisterNewMember() {
            MemberStorage.loadMembers();
            setIconImage(icon.getImage());
            setTitle("New Member Registration");
            setSize(600, 500);
            setDefaultCloseOperation(EXIT_ON_CLOSE);
            setLayout(new BorderLayout());

            JPanel topPanel = new JPanel();
            topPanel.setBackground(new Color(169, 169, 169));
            JLabel title = new JLabel("New Member Registration", SwingConstants.CENTER);
            title.setFont(new Font("Arial", Font.BOLD, 24));
            title.setForeground(Color.WHITE);
            topPanel.add(title);
            add(topPanel, BorderLayout.NORTH);

            JPanel formPanel = new JPanel(new GridLayout(9, 2, 10, 10));
            formPanel.setBackground(new Color(220, 220, 220));
            formPanel.setBorder(BorderFactory.createEmptyBorder(20, 40, 20, 40));

            JTextField nameField = new JTextField();
            JTextField ageField = new JTextField();
            JComboBox<String> genderField = new JComboBox<>(new String[]{"Male", "Female", "Other"});
            JTextField phoneField = new JTextField();
            JTextField emailField = new JTextField();

            String[] plans = {"Pay Per Visit", "Monthly", "3 Months", "6 Months", "Yearly"};
            JComboBox<String> planBox = new JComboBox<>(plans);
            JLabel planDetails = new JLabel("Select a plan to view details");

            Map<String, String> planInfo = Map.of(
                    "Pay Per Visit", "50 EGP/visit. No commitment.",
                    "Monthly", "500 EGP/month. Includes classes.",
                    "3 Months", "1300 EGP. Includes trainer access.",
                    "6 Months", "2500 EGP. Trainer + nutrition plan.",
                    "Yearly", "4500 EGP. All benefits included."
            );

            planBox.addActionListener(e ->
                    planDetails.setText(planInfo.get(planBox.getSelectedItem()))
            );

            RoundedButton registerBtn = new RoundedButton("Register");
            RoundedButton backBtn = new RoundedButton("Cancel");

            // Form layout
            formPanel.add(new JLabel("Full Name:"));
            formPanel.add(nameField);
            formPanel.add(new JLabel("Age:"));
            formPanel.add(ageField);
            formPanel.add(new JLabel("Gender:"));
            formPanel.add(genderField);
            formPanel.add(new JLabel("Phone Number:"));
            formPanel.add(phoneField);
            formPanel.add(new JLabel("Email:"));
            formPanel.add(emailField);
            formPanel.add(new JLabel("Membership Plan:"));
            formPanel.add(planBox);
            formPanel.add(new JLabel("Plan Details:"));
            formPanel.add(planDetails);
            formPanel.add(backBtn);
            formPanel.add(registerBtn);

            registerBtn.addActionListener(e -> registerMember(
                    nameField.getText().trim(),
                    ageField.getText().trim(),
                    (String) genderField.getSelectedItem(),
                    phoneField.getText().trim(),
                    emailField.getText().trim(),
                    (String) planBox.getSelectedItem()
            ));

            backBtn.addActionListener(e -> {
                dispose(); // Close current window
                new MainPage();
            });

            add(formPanel, BorderLayout.CENTER);
            setLocationRelativeTo(null);
            setVisible(true);
        }

        private void registerMember(String name, String ageText, String gender,
                                  String phoneText, String email, String plan) {
            // Validation
            if (name.isEmpty() || ageText.isEmpty() || phoneText.isEmpty() || email.isEmpty()) {
                showError("Please fill in all fields");
                return;
            }

            int age, phonenumber;
            try {
                age = Integer.parseInt(ageText);
                phonenumber = Integer.parseInt(phoneText);
            } catch (NumberFormatException e) {
                showError("Age and Phone must be valid numbers");
                return;
            }

            if (!email.contains("@") || !email.contains(".")) {
                showError("Please enter a valid email address");
                return;
            }

            // Calculate dates based on plan
            LocalDate startDate = LocalDate.now();
            LocalDate endDate = calculateEndDate(plan, startDate);

            // Create member
            String id = "M" + (MemberStorage.members.size() + 1000);
            Member newMember = new Member(
                    id, name, age, gender, email, plan,
                    phonenumber, startDate, endDate
            );

            // Save and log
            MemberStorage.members.add(newMember);
            MemberStorage.saveMembers();
            ActivityLogger.logActivity(
                    "New registration: " + name + " (" + id + ") - " + plan
            );

            // Show success
            showSuccess(newMember);
        }

        private LocalDate calculateEndDate(String plan, LocalDate startDate) {
            switch (plan) {
                case "Monthly":
                    return startDate.plusMonths(1);
                case "3 Months":
                    return startDate.plusMonths(3);
                case "6 Months":
                    return startDate.plusMonths(6);
                case "Yearly":
                    return startDate.plusYears(1);
                default:
                    return startDate; // Pay Per Visit
            }
        }

        private void showError(String message) {
            JOptionPane.showMessageDialog(this,
                    message,
                    "Registration Error",
                    JOptionPane.ERROR_MESSAGE);
        }

        private void showSuccess(Member member) {
            String[] columns = {"Field", "Value"};
            Object[][] data = {
                    {"Member ID", member.id},
                    {"Name", member.name},
                    {"Plan", member.plan},
                    {"Start Date", member.startdate},
                    {"End Date", member.enddate},
                    {"Status", "Active"}
            };

            JTable table = new JTable(data, columns);
            table.setFillsViewportHeight(true);

            JPanel panel = new JPanel(new BorderLayout());
            panel.add(new JLabel("Registration Successful!"), BorderLayout.NORTH);
            panel.add(new JScrollPane(table), BorderLayout.CENTER);

            JOptionPane.showMessageDialog(this, panel,
                    "Registration Complete",
                    JOptionPane.INFORMATION_MESSAGE);

            dispose();
            new MainPage();
        }
    }

    // ---------- Main Page ----------
    static class MainPage {
        private JFrame frame;
        private ImageIcon icon = new ImageIcon("weight.png");

        public MainPage() {
            MemberStorage.loadMembers();
            frame = new JFrame("Fitness Gym Manager");
            frame.setIconImage(icon.getImage());
            frame.setSize(650, 550);
            frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            frame.setLayout(new BorderLayout());

            JLabel title = new JLabel("Fitness Gym Manager", JLabel.CENTER);
            title.setFont(new Font("Arial", Font.BOLD, 30));
            title.setForeground(new Color(33, 33, 33));
            frame.add(title, BorderLayout.NORTH);

            JPanel buttonPanel = new JPanel(new GridLayout(3, 1, 25, 25));
            buttonPanel.setBackground(Color.WHITE);
            buttonPanel.setBorder(BorderFactory.createEmptyBorder(40, 40, 40, 40));

            RoundedButton adminBtn = new RoundedButton("Admin Dashboard");
            RoundedButton loginBtn = new RoundedButton("Member Login");
            RoundedButton registerBtn = new RoundedButton("Register Member");

            // Ensure admin button properly handles window management
            adminBtn.addActionListener(e -> {
                frame.dispose(); // Close main page
                new AdminDashboardLogin(); // Open admin login
            });
            
            loginBtn.addActionListener(e -> {
                frame.dispose(); // Close current window
                new LoginPage();
            });
            
            registerBtn.addActionListener(e -> {
                frame.dispose(); // Close current window
                new RegisterNewMember();
            });

            buttonPanel.add(adminBtn);
            buttonPanel.add(loginBtn);
            buttonPanel.add(registerBtn);

            frame.add(buttonPanel, BorderLayout.CENTER);
            frame.setLocationRelativeTo(null);
            frame.setVisible(true);
        }
    }

    // ---------- Admin Login ----------
    static class AdminDashboardLogin extends JFrame {
        private ImageIcon icon = new ImageIcon("weight.png");

        public AdminDashboardLogin() {
            Map<String, String> adminAccounts = Map.of(
                    "Omar", "omar123",
                    "Ahmed", "ahmed456",
                    "Abdelmawla", "abdelmawla789",
                    "Abdelrahman", "abdelrahman012",
                    "Hamza", "hamza345"
            );

            setTitle("Admin Login");
            setIconImage(icon.getImage());
            setSize(400, 300);
            setDefaultCloseOperation(EXIT_ON_CLOSE);
            setLayout(new BorderLayout());

            JPanel topPanel = new JPanel();
            topPanel.setBackground(new Color(169, 169, 169));
            JLabel title = new JLabel("Admin Login", SwingConstants.CENTER);
            title.setFont(new Font("Arial", Font.BOLD, 20));
            title.setForeground(Color.WHITE);
            topPanel.add(title);
            add(topPanel, BorderLayout.NORTH);

            JPanel centerPanel = new JPanel(new GridLayout(3, 2, 10, 10));
            centerPanel.setBackground(new Color(220, 220, 220));
            centerPanel.setBorder(BorderFactory.createEmptyBorder(20, 20, 20, 20));

            JLabel userLabel = new JLabel("Username:");
            JLabel passLabel = new JLabel("Password:");
            JTextField userField = new JTextField();
            JPasswordField passField = new JPasswordField();
            RoundedButton loginBtn = new RoundedButton("Login");
            RoundedButton backBtn = new RoundedButton("Back");

            centerPanel.add(userLabel);
            centerPanel.add(userField);
            centerPanel.add(passLabel);
            centerPanel.add(passField);
            centerPanel.add(loginBtn);
            centerPanel.add(backBtn);

            loginBtn.addActionListener(e -> {
                String username = userField.getText();
                String password = new String(passField.getPassword());
                
                if (adminAccounts.containsKey(username) && 
                    adminAccounts.get(username).equals(password)) {
                    dispose(); // Close current window
                    new AdminDashboard();
                }
            });
            
            backBtn.addActionListener(e -> {
                dispose(); // Close current window
                new MainPage();
            });

            add(centerPanel, BorderLayout.CENTER);
            setLocationRelativeTo(null);
            setVisible(true);
        }
    }

    // ---------- Admin Dashboard ----------
    static class AdminDashboard extends JFrame {
        private ImageIcon icon = new ImageIcon("weight.png");

        public AdminDashboard() {
            MemberStorage.loadMembers();
            setIconImage(icon.getImage());
            setTitle("Admin Dashboard");
            setSize(800, 600);
            setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            setLayout(new BorderLayout());

            JPanel topPanel = new JPanel();
            topPanel.setBackground(new Color(169, 169, 169));
            JLabel title = new JLabel("Admin Dashboard", SwingConstants.CENTER);
            title.setFont(new Font("Arial", Font.BOLD, 24));
            title.setForeground(Color.WHITE);
            topPanel.add(title);
            add(topPanel, BorderLayout.NORTH);

            JPanel buttonPanel = new JPanel(new GridLayout(4, 2, 15, 15));
            buttonPanel.setBackground(new Color(220, 220, 220));
            buttonPanel.setBorder(BorderFactory.createEmptyBorder(20, 20, 20, 20));

            RoundedButton viewMembersBtn = new RoundedButton("View/Edit/Delete Clients");
            RoundedButton activeInactiveBtn = new RoundedButton("Active/Inactive Members");
            RoundedButton attendanceBtn = new RoundedButton("Attendance");
            RoundedButton revenueBtn = new RoundedButton("Revenue");
            RoundedButton overdueBtn = new RoundedButton("Overdue Members");
            RoundedButton reportsBtn = new RoundedButton("Generate Reports");
            RoundedButton frequentBtn = new RoundedButton("Frequent Visitors");
            RoundedButton backBtn = new RoundedButton("Back to Main");

            viewMembersBtn.addActionListener(e -> {
                dispose(); // Close current window
                new MemberManagerPage();
            });
            
            activeInactiveBtn.addActionListener(e -> {
                dispose(); // Close current window
                new ActiveInactiveMembersPage();
            });

            attendanceBtn.addActionListener(e -> {
                dispose(); // Close current window
                new AttendanceHistoryPage();
            });

            reportsBtn.addActionListener(e -> {
                dispose(); // Close current window
                ReportsGenerator.generateMonthlyReport();
            new AdminDashboard(); // Reopen dashboard after report closes
            });

            revenueBtn.addActionListener(e -> {
                dispose();
                new RevenueReportDialog(AdminDashboard.this);
            });
            
            backBtn.addActionListener(e -> {
                dispose(); // Close current window
                new MainPage();
            });

            revenueBtn.addActionListener(e -> showPlaceholder("Revenue"));
            overdueBtn.addActionListener(e -> showPlaceholder("Overdue Members"));
            frequentBtn.addActionListener(e -> showPlaceholder("Frequent Visitors"));

            buttonPanel.add(viewMembersBtn);
            buttonPanel.add(activeInactiveBtn);
            buttonPanel.add(attendanceBtn);
            buttonPanel.add(revenueBtn);
            buttonPanel.add(overdueBtn);
            buttonPanel.add(reportsBtn);
            buttonPanel.add(frequentBtn);
            buttonPanel.add(backBtn);

            add(buttonPanel, BorderLayout.CENTER);
            setLocationRelativeTo(null);
            setVisible(true);
        }

        private void showPlaceholder(String feature) {
            JOptionPane.showMessageDialog(this,
                    feature + " feature will be implemented in next version",
                    "Coming Soon",
                    JOptionPane.INFORMATION_MESSAGE);
        }
    }

    // ---------- Active/Inactive Members Page ----------
    static class ActiveInactiveMembersPage extends JFrame {
        private ImageIcon icon = new ImageIcon("weight.png");

        public ActiveInactiveMembersPage() {
            MemberStorage.loadMembers();
            setIconImage(icon.getImage());
            setTitle("Active/Inactive Members");
            setSize(800, 600);
            setDefaultCloseOperation(EXIT_ON_CLOSE);
            setLayout(new BorderLayout());

            JPanel topPanel = new JPanel();
            topPanel.setBackground(new Color(169, 169, 169));
            JLabel title = new JLabel("Active/Inactive Members", SwingConstants.CENTER);
            title.setFont(new Font("Arial", Font.BOLD, 20));
            title.setForeground(Color.WHITE);
            topPanel.add(title);
            add(topPanel, BorderLayout.NORTH);

            JTabbedPane tabbedPane = new JTabbedPane();

            // Active Members Tab
            JTable activeTable = createMemberTable(MemberStorage.getActiveMembers(), "Active");
            tabbedPane.addTab("Active Members", new JScrollPane(activeTable));

            // Inactive Members Tab
            JTable inactiveTable = createMemberTable(MemberStorage.getInactiveMembers(), "Inactive");
            tabbedPane.addTab("Inactive Members", new JScrollPane(inactiveTable));

            add(tabbedPane, BorderLayout.CENTER);

            RoundedButton backBtn = new RoundedButton("Back to Dashboard");
            backBtn.addActionListener(e -> {
                dispose();
                new AdminDashboard();
            });

            JPanel bottomPanel = new JPanel();
            bottomPanel.add(backBtn);
            add(bottomPanel, BorderLayout.SOUTH);

            setLocationRelativeTo(null);
            setVisible(true);

            backBtn.addActionListener(e -> {
                dispose(); // Close current window
                new AdminDashboard();
            });
        }

        private JTable createMemberTable(List<Member> members, String status) {
            String[] columns = {"ID", "Name", "Plan", "Start Date", "End Date", "Status"};
            DefaultTableModel model = new DefaultTableModel(columns, 0) {
                @Override
                public boolean isCellEditable(int row, int column) {
                    return false;
                }
            };

            members.forEach(m ->
                    model.addRow(new Object[]{
                            m.id, m.name, m.plan,
                            m.startdate, m.enddate, status
                    })
            );

            JTable table = new JTable(model);
            table.setAutoCreateRowSorter(true);
            table.setFillsViewportHeight(true);

            // Custom column sizing
            table.getColumnModel().getColumn(0).setPreferredWidth(80);  // ID
            table.getColumnModel().getColumn(1).setPreferredWidth(150); // Name
            table.getColumnModel().getColumn(2).setPreferredWidth(100); // Plan
            table.getColumnModel().getColumn(3).setPreferredWidth(100); // Start Date
            table.getColumnModel().getColumn(4).setPreferredWidth(100); // End Date
            table.getColumnModel().getColumn(5).setPreferredWidth(80);  // Status

            return table;
        }
    }

    // ---------- Member Manager Page ----------
    static class MemberManagerPage extends JFrame {
        public MemberManagerPage() {
            setTitle("View/Edit/Delete Clients");
            setSize(800, 600);
            setLayout(new BorderLayout());
            MemberStorage.loadMembers();

            // Top panel with title
            JPanel topPanel = new JPanel();
            topPanel.setBackground(new Color(169, 169, 169));
            JLabel title = new JLabel("Client Management", SwingConstants.CENTER);
            title.setFont(new Font("Arial", Font.BOLD, 24));
            title.setForeground(Color.WHITE);
            topPanel.add(title);
            add(topPanel, BorderLayout.NORTH);

            // Table of all members
            String[] columnNames = {"ID", "Name", "Age", "Gender", "Email", "Phone", "Plan", "Start Date", "End Date", "Status"};
            Object[][] data = new Object[MemberStorage.members.size()][10];

            for (int i = 0; i < MemberStorage.members.size(); i++) {
                Member m = MemberStorage.members.get(i);
                data[i][0] = m.id;
                data[i][1] = m.name;
                data[i][2] = m.age;
                data[i][3] = m.gender;
                data[i][4] = m.email;
                data[i][5] = m.phonenumber;
                data[i][6] = m.plan;
                data[i][7] = m.startdate;
                data[i][8] = m.enddate;
                data[i][9] = m.enddate.isBefore(LocalDate.now()) ? "Inactive" : "Active";
            }

            JTable table = new JTable(data, columnNames);
            table.setAutoCreateRowSorter(true);
            JScrollPane scrollPane = new JScrollPane(table);
            scrollPane.setPreferredSize(new Dimension(750, 400));
            add(scrollPane, BorderLayout.CENTER);

            // Button panel (only Edit, Delete, Back)
            JPanel buttonPanel = new JPanel(new FlowLayout(FlowLayout.CENTER, 20, 10));
            buttonPanel.setBackground(new Color(220, 220, 220));

            RoundedButton editBtn = new RoundedButton("Edit Member");
            RoundedButton deleteBtn = new RoundedButton("Delete Member");
            RoundedButton backBtn = new RoundedButton("Back to Dashboard");

            editBtn.addActionListener(e -> {
                int selectedRow = table.getSelectedRow();
                if (selectedRow < 0) {
                    JOptionPane.showMessageDialog(this, "Please select a member to edit", "No Selection", JOptionPane.WARNING_MESSAGE);
                    return;
                }

                String id = (String) table.getValueAt(selectedRow, 0);
                String name = JOptionPane.showInputDialog("New name:", table.getValueAt(selectedRow, 1));
                String email = JOptionPane.showInputDialog("New email:", table.getValueAt(selectedRow, 4));
                String phone = JOptionPane.showInputDialog("New phone:", table.getValueAt(selectedRow, 5));

                boolean edited = MemberStorage.editMember(id, name, email, phone);
                JOptionPane.showMessageDialog(this, edited ? " Edited" : " ID not found");
                dispose();
                new MemberManagerPage(); // Refresh the view
            });

            deleteBtn.addActionListener(e -> {
                int selectedRow = table.getSelectedRow();
                if (selectedRow < 0) {
                    JOptionPane.showMessageDialog(this, "Please select a member to delete", "No Selection", JOptionPane.WARNING_MESSAGE);
                    return;
                }

                String id = (String) table.getValueAt(selectedRow, 0);
                String name = (String) table.getValueAt(selectedRow, 1);

                int confirm = JOptionPane.showConfirmDialog(
                        this,
                        "Are you sure you want to delete " + name + "?",
                        "Confirm Deletion",
                        JOptionPane.YES_NO_OPTION);

                if (confirm == JOptionPane.YES_OPTION) {
                    boolean removed = MemberStorage.removeMember(id);
                    JOptionPane.showMessageDialog(this, removed ? " Deleted" : " ID not found");
                    dispose();
                    new MemberManagerPage(); // Refresh the view
                }
            });

            backBtn.addActionListener(e -> {
                dispose(); // Close current window
                new AdminDashboard();
            });

            buttonPanel.add(editBtn);
            buttonPanel.add(deleteBtn);
            buttonPanel.add(backBtn);
            add(buttonPanel, BorderLayout.SOUTH);

            setLocationRelativeTo(null);
            setVisible(true);
        }
    }

    // ---------- Member Dashboard ----------
    static class MemberDashboard extends JFrame {
        private ImageIcon icon = new ImageIcon("weight.png");
        private Member member;

        public MemberDashboard(Member member) {
            this.member = member;
            setIconImage(icon.getImage());
            setTitle("Member Dashboard - " + member.name);
            setSize(500, 400);
            setDefaultCloseOperation(EXIT_ON_CLOSE);
            setLayout(new BorderLayout());

            JPanel topPanel = new JPanel();
            topPanel.setBackground(new Color(169, 169, 169));
            JLabel welcome = new JLabel("Welcome, " + member.name, SwingConstants.CENTER);
            welcome.setFont(new Font("Arial", Font.BOLD, 24));
            welcome.setForeground(Color.WHITE);
            topPanel.add(welcome);
            add(topPanel, BorderLayout.NORTH);

            JPanel centerPanel = new JPanel(new GridLayout(4, 1, 15, 15));
            centerPanel.setBackground(new Color(220, 220, 220));
            centerPanel.setBorder(BorderFactory.createEmptyBorder(30, 30, 30, 30));

            RoundedButton checkInBtn = new RoundedButton("Check In");
            RoundedButton profileBtn = new RoundedButton("View Profile");
            RoundedButton attendanceBtn = new RoundedButton("Attendance History");
            RoundedButton backBtn = new RoundedButton("Logout");

            checkInBtn.addActionListener(e -> {
                String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
                ActivityLogger.logActivity("Check-in: " + member.name + " (" + member.id + ") at " + timestamp);
                JOptionPane.showMessageDialog(this,
                        "Checked in successfully at " + timestamp,
                        "Check-In Confirmed",
                        JOptionPane.INFORMATION_MESSAGE);
            });

            profileBtn.addActionListener(e -> showProfile());
            attendanceBtn.addActionListener(e -> showAttendanceHistory());
            backBtn.addActionListener(e -> {
                dispose(); // Close current window
                new MainPage();
            });

            centerPanel.add(checkInBtn);
            centerPanel.add(profileBtn);
            centerPanel.add(attendanceBtn);
            centerPanel.add(backBtn);

            add(centerPanel, BorderLayout.CENTER);
            setLocationRelativeTo(null);
            setVisible(true);
        }

        private void showProfile() {
            String[] columns = {"Field", "Value"};
            Object[][] data = {
                    {"Member ID", member.id},
                    {"Name", member.name},
                    {"Age", member.age},
                    {"Gender", member.gender},
                    {"Email", member.email},
                    {"Phone", member.phonenumber},
                    {"Membership Plan", member.plan},
                    {"Start Date", member.startdate},
                    {"End Date", member.enddate},
                    {"Status", member.enddate.isBefore(LocalDate.now()) ? "Inactive" : "Active"}
            };

            JTable table = new JTable(data, columns);
            table.setFillsViewportHeight(true);
            JScrollPane scrollPane = new JScrollPane(table);
            scrollPane.setPreferredSize(new Dimension(500, 300));

            JOptionPane.showMessageDialog(this, scrollPane, "Your Profile", JOptionPane.PLAIN_MESSAGE);
        }

        private void showAttendanceHistory() {
            List<String[]> activities = ActivityLogger.getRecentActivities(20).stream()
                    .filter(a -> a[1].contains(member.id))
                    .map(a -> new String[]{
                            a[0].split("T")[0], // Date
                            a[0].split("T")[1].substring(0, 8), // Time
                            a[1].replace("Check-in: " + member.name + " (" + member.id + ") at ", "")
                    })
                    .collect(Collectors.toList());

            String[] columns = {"Date", "Time", "Activity"};
            DefaultTableModel model = new DefaultTableModel(columns, 0) {
                @Override
                public boolean isCellEditable(int row, int column) {
                    return false;
                }
            };

            activities.forEach(model::addRow);

            JTable table = new JTable(model);
            table.setAutoCreateRowSorter(true);
            JScrollPane scrollPane = new JScrollPane(table);
            scrollPane.setPreferredSize(new Dimension(600, 400));

            JOptionPane.showMessageDialog(this, scrollPane, "Your Attendance History", JOptionPane.PLAIN_MESSAGE);
        }
    }
// ---------- Attendance History Page ----------
static class AttendanceHistoryPage extends JFrame {
    private ImageIcon icon = new ImageIcon("weight.png");

    public AttendanceHistoryPage() {
        setIconImage(icon.getImage());
        setTitle("Attendance History");
        setSize(900, 600);
        setDefaultCloseOperation(DISPOSE_ON_CLOSE);
        setLayout(new BorderLayout());

        // Top panel with title
        JPanel topPanel = new JPanel();
        topPanel.setBackground(new Color(169, 169, 169));
        JLabel title = new JLabel("Attendance History", SwingConstants.CENTER);
        title.setFont(new Font("Arial", Font.BOLD, 24));
        title.setForeground(Color.WHITE);
        topPanel.add(title);
        add(topPanel, BorderLayout.NORTH);

        // Get all activities and filter for check-ins
        List<String[]> allActivities = ActivityLogger.getRecentActivities(Integer.MAX_VALUE);
        List<String[]> checkIns = allActivities.stream()
                .filter(a -> a[1].contains("Check-in:"))
                .collect(Collectors.toList());

        // Prepare table data
        String[] columns = {"Date", "Time", "Member ID", "Member Name", "Check-in Time"};
        DefaultTableModel model = new DefaultTableModel(columns, 0) {
            @Override
            public boolean isCellEditable(int row, int column) {
                return false;
            }
        };

        // Parse and add each check-in to the table
        for (String[] activity : checkIns) {
            String[] dateTime = activity[0].split("T");
            String date = dateTime[0];
            String time = dateTime[1].substring(0, 8);
            
            // Extract member info from activity log
            String activityText = activity[1];
            String memberId = activityText.substring(
                activityText.indexOf("(") + 1,
                activityText.indexOf(")")
            );
            String memberName = activityText.substring(
                "Check-in: ".length(),
                activityText.indexOf(" (")
            );
            String checkInTime = activityText.substring(
                activityText.indexOf("at ") + 3
            );

            model.addRow(new Object[]{date, time, memberId, memberName, checkInTime});
        }

        // Create and configure table
        JTable table = new JTable(model);
        table.setAutoCreateRowSorter(true);
        table.setFillsViewportHeight(true);
        
        // Set column widths
        table.getColumnModel().getColumn(0).setPreferredWidth(100); // Date
        table.getColumnModel().getColumn(1).setPreferredWidth(80);  // Time
        table.getColumnModel().getColumn(2).setPreferredWidth(100); // Member ID
        table.getColumnModel().getColumn(3).setPreferredWidth(150); // Member Name
        table.getColumnModel().getColumn(4).setPreferredWidth(150); // Check-in Time

        JScrollPane scrollPane = new JScrollPane(table);
        add(scrollPane, BorderLayout.CENTER);

        // Bottom panel with buttons
        JPanel bottomPanel = new JPanel(new FlowLayout(FlowLayout.CENTER, 20, 10));
        bottomPanel.setBackground(new Color(220, 220, 220));
        
        RoundedButton refreshBtn = new RoundedButton("Refresh");
        RoundedButton backBtn = new RoundedButton("Back to Dashboard");

        refreshBtn.addActionListener(e -> {
            dispose();
            new AttendanceHistoryPage();
        });

        backBtn.addActionListener(e -> {
            dispose();
            new AdminDashboard();
        });

        bottomPanel.add(refreshBtn);
        bottomPanel.add(backBtn);
        add(bottomPanel, BorderLayout.SOUTH);

        setLocationRelativeTo(null);
        setVisible(true);
    }
}

// ---------- revenue analytics ----------
static class RevenueReportDialog extends JDialog {
    public RevenueReportDialog(JFrame parent) {
        super(parent, "Revenue Analytics", true);
        
        // Calculate revenue based on your actual membership plans
        Map<String, Integer> planCounts = MemberStorage.members.stream()
            .collect(Collectors.groupingBy(m -> m.plan, Collectors.summingInt(m -> 1)));
        
        // Use the same prices as in your RegisterNewMember class
        double payPerVisit = planCounts.getOrDefault("Pay Per Visit", 0) * 50;
        double monthly = planCounts.getOrDefault("Monthly", 0) * 500;
        double threeMonths = planCounts.getOrDefault("3 Months", 0) * 1300;
        double sixMonths = planCounts.getOrDefault("6 Months", 0) * 2500;
        double yearly = planCounts.getOrDefault("Yearly", 0) * 4500;
        
        double totalRevenue = payPerVisit + monthly + threeMonths + sixMonths + yearly;
        double tax = totalRevenue * 0.14;
        double expenses = totalRevenue * 0.2;
        double netProfit = totalRevenue - tax - expenses;
        
        // Create the display
        String[] columns = {"Plan", "Subscribers", "Revenue"};
        Object[][] data = {
            {"Pay Per Visit", planCounts.getOrDefault("Pay Per Visit", 0), payPerVisit},
            {"Monthly", planCounts.getOrDefault("Monthly", 0), monthly},
            {"3 Months", planCounts.getOrDefault("3 Months", 0), threeMonths},
            {"6 Months", planCounts.getOrDefault("6 Months", 0), sixMonths},
            {"Yearly", planCounts.getOrDefault("Yearly", 0), yearly}
        };
        
        JTable table = new JTable(data, columns);
        table.setEnabled(false);
        
        JPanel summaryPanel = new JPanel(new GridLayout(3, 1));
        summaryPanel.add(new JLabel(String.format("Total Revenue: %.2f EGP", totalRevenue)));
        summaryPanel.add(new JLabel(String.format("Total Deductions: %.2f EGP", (tax + expenses))));
        summaryPanel.add(new JLabel(String.format("Net Profit: %.2f EGP", netProfit)));
        
        setLayout(new BorderLayout());
        add(new JScrollPane(table), BorderLayout.CENTER);
        add(summaryPanel, BorderLayout.SOUTH);
        
        setSize(500, 400);
        setLocationRelativeTo(parent);
        setVisible(true);
    }
}
}
