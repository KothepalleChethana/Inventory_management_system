import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.sql.*;

public class InventoryManagementSystemUI {
    private static final String JDBC_URL = "jdbc:mysql://localhost:3306/inventory_db";
    private static final String DB_USER = "root";
    private static final String DB_PASSWORD = "charitha13";

    private JFrame frame;
    private JTextField usernameField;
    private JPasswordField passwordField;
    private JTextArea outputTextArea;
    private User currentUser;

    public static void main(String[] args) {
        EventQueue.invokeLater(() -> {
            try {
                InventoryManagementSystemUI window = new InventoryManagementSystemUI();
                window.frame.setVisible(true);
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
    }

    public InventoryManagementSystemUI() {
        initialize();
    }

    private void initialize() {
        frame = new JFrame();
        frame.setTitle("Inventory Management System");
        frame.setBounds(100, 100, 600, 400);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.getContentPane().setLayout(new CardLayout(0, 0));

        JPanel loginPanel = createLoginPanel();
        frame.getContentPane().add(loginPanel, "loginPanel");

        JPanel mainPanel = createMainPanel();
        frame.getContentPane().add(mainPanel, "mainPanel");

        switchToLoginPanel();
    }

    private JPanel createLoginPanel() {
        JPanel panel = new JPanel();
        panel.setLayout(null);

        JLabel lblNewLabel = new JLabel("Username:");
        lblNewLabel.setBounds(150, 100, 100, 20);
        panel.add(lblNewLabel);

        usernameField = new JTextField();
        usernameField.setBounds(250, 100, 150, 20);
        panel.add(usernameField);
        usernameField.setColumns(10);

        JLabel lblPassword = new JLabel("Password:");
        lblPassword.setBounds(150, 150, 100, 20);
        panel.add(lblPassword);

        passwordField = new JPasswordField();
        passwordField.setBounds(250, 150, 150, 20);
        panel.add(passwordField);

        JButton loginButton = new JButton("Login");
        loginButton.addActionListener(e -> login());
        loginButton.setBounds(250, 200, 100, 30);
        panel.add(loginButton);

        return panel;
    }

    private JPanel createMainPanel() {
        JPanel panel = new JPanel();
        panel.setLayout(null);

        outputTextArea = new JTextArea();
        outputTextArea.setEditable(false);
        JScrollPane scrollPane = new JScrollPane(outputTextArea);
        scrollPane.setBounds(50, 50, 500, 200);
        panel.add(scrollPane);

        JButton viewInventoryButton = new JButton("View Inventory");
        viewInventoryButton.addActionListener(e -> viewInventory());
        viewInventoryButton.setBounds(50, 300, 150, 30);
        panel.add(viewInventoryButton);

        JButton addItemButton = new JButton("Add Item");
        addItemButton.addActionListener(e -> addItem());
        addItemButton.setBounds(250, 300, 150, 30);
        panel.add(addItemButton);

        JButton updateItemButton = new JButton("Update Item");
        updateItemButton.addActionListener(e -> updateItem());
        updateItemButton.setBounds(450, 300, 150, 30);
        panel.add(updateItemButton);

        JButton deleteItemButton = new JButton("Delete Item");
        deleteItemButton.addActionListener(e -> deleteItem());
        deleteItemButton.setBounds(250, 350, 150, 30);
        panel.add(deleteItemButton);

        JButton logoutButton = new JButton("Logout");
        logoutButton.addActionListener(e -> switchToLoginPanel());
        logoutButton.setBounds(450, 350, 100, 30);
        panel.add(logoutButton);

        return panel;
    }

    private void login() {
        String username = usernameField.getText();
        String password = new String(passwordField.getPassword());

        User user = authenticateUser(username, password);

        if (user != null) {
            currentUser = user;
            switchToMainPanel();
            outputTextArea.append("Login successful!\n");
            displayWelcomeMessage();
        } else {
            outputTextArea.append("Invalid username or password. Login failed.\n");
        }

        // Clear the password field
        passwordField.setText("");
    }

    private void displayWelcomeMessage() {
        if (currentUser != null) {
            String welcomeMessage = "Welcome ";
            if (currentUser.role.equals("owner")) {
                welcomeMessage += "Owner!!";
            } else {
                welcomeMessage += "Employee!!";
            }
            outputTextArea.setText(welcomeMessage + "\n\n");
        }
    }

    private void switchToLoginPanel() {
        CardLayout cardLayout = (CardLayout) frame.getContentPane().getLayout();
        cardLayout.show(frame.getContentPane(), "loginPanel");
        outputTextArea.setText(""); // Clear output text area
    }

    private void switchToMainPanel() {
        CardLayout cardLayout = (CardLayout) frame.getContentPane().getLayout();
        cardLayout.show(frame.getContentPane(), "mainPanel");
        viewInventory(); // Display initial inventory on login
    }

    private User authenticateUser(String username, String password) {
        try (Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD)) {
            String sql = "SELECT username, password, role FROM users WHERE username = ? AND password = ?";
            try (PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
                preparedStatement.setString(1, username);
                preparedStatement.setString(2, password);
                try (ResultSet resultSet = preparedStatement.executeQuery()) {
                    if (resultSet.next()) {
                        String role = resultSet.getString("role");
                        return new User(username, password, role);
                    }
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
            outputTextArea.append("Error authenticating user.\n");
        }
        return null;
    }

    private void viewInventory() {
        outputTextArea.setText(""); // Clear output text area
        try (Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD)) {
            String sql = "SELECT item_name, quantity, price FROM inventory";
            try (PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
                try (ResultSet resultSet = preparedStatement.executeQuery()) {
                    while (resultSet.next()) {
                        String itemName = resultSet.getString("item_name");
                        int quantity = resultSet.getInt("quantity");
                        double price = resultSet.getDouble("price");
                        outputTextArea.append(String.format("%-20s%-10d%-15.2f%n", itemName, quantity, price));
                    }
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
            outputTextArea.append("Error retrieving inventory.\n");
        }
    }

    private void addItem() {
        if (currentUser != null && currentUser.role.equals("owner")) {
            String itemName = JOptionPane.showInputDialog(frame, "Enter item name:");
            String quantityStr = JOptionPane.showInputDialog(frame, "Enter quantity:");
            String priceStr = JOptionPane.showInputDialog(frame, "Enter price:");

            try (Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD)) {
                String sql = "INSERT INTO inventory (item_name, quantity, price) VALUES (?, ?, ?)";
                try (PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
                    preparedStatement.setString(1, itemName);
                    preparedStatement.setInt(2, Integer.parseInt(quantityStr));
                    preparedStatement.setDouble(3, Double.parseDouble(priceStr));
                    preparedStatement.executeUpdate();
                    outputTextArea.append("Item added to inventory successfully!\n");
                }
            } catch (SQLException e) {
                e.printStackTrace();
                outputTextArea.append("Error adding item to inventory.\n");
            }
        } else {
            outputTextArea.append("You do not have permission to perform this operation.\n");
        }
    }

    private void updateItem() {
        if (currentUser != null && currentUser.role.equals("owner")) {
            String itemNameToUpdate = JOptionPane.showInputDialog(frame, "Enter item name to update:");
            try (Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD)) {
                String selectSql = "SELECT * FROM inventory WHERE item_name = ?";
                try (PreparedStatement selectStatement = connection.prepareStatement(selectSql)) {
                    selectStatement.setString(1, itemNameToUpdate);
                    try (ResultSet resultSet = selectStatement.executeQuery()) {
                        if (resultSet.next()) {
                            int currentQuantity = resultSet.getInt("quantity");
                            double currentPrice = resultSet.getDouble("price");

                            String newQuantityStr = JOptionPane.showInputDialog(frame, "Enter new quantity (current: " + currentQuantity + "):");
                            String newPriceStr = JOptionPane.showInputDialog(frame, "Enter new price (current: " + currentPrice + "):");

                            String updateSql = "UPDATE inventory SET quantity = ?, price = ? WHERE item_name = ?";
                            try (PreparedStatement updateStatement = connection.prepareStatement(updateSql)) {
                                updateStatement.setInt(1, Integer.parseInt(newQuantityStr));
                                updateStatement.setDouble(2, Double.parseDouble(newPriceStr));
                                updateStatement.setString(3, itemNameToUpdate);
                                updateStatement.executeUpdate();
                                outputTextArea.append("Item updated successfully!\n");
                            }
                        } else {
                            outputTextArea.append("Item not found in inventory. Update failed.\n");
                        }
                    }
                }
            } catch (SQLException e) {
                e.printStackTrace();
                outputTextArea.append("Error updating item in inventory.\n");
            }
        } else {
            outputTextArea.append("You do not have permission to perform this operation.\n");
        }
    }

    private void deleteItem() {
        if (currentUser != null && currentUser.role.equals("owner")) {
            String itemNameToDelete = JOptionPane.showInputDialog(frame, "Enter item name to delete:");
            try (Connection connection = DriverManager.getConnection(JDBC_URL, DB_USER, DB_PASSWORD)) {
                String selectSql = "SELECT * FROM inventory WHERE item_name = ?";
                try (PreparedStatement selectStatement = connection.prepareStatement(selectSql)) {
                    selectStatement.setString(1, itemNameToDelete);
                    try (ResultSet resultSet = selectStatement.executeQuery()) {
                        if (resultSet.next()) {
                            String deleteSql = "DELETE FROM inventory WHERE item_name = ?";
                            try (PreparedStatement deleteStatement = connection.prepareStatement(deleteSql)) {
                                deleteStatement.setString(1, itemNameToDelete);
                                deleteStatement.executeUpdate();
                                outputTextArea.append("Item deleted successfully!\n");
                            }
                        } else {
                            outputTextArea.append("Item not found in inventory. Deletion failed.\n");
                        }
                    }
                }
            } catch (SQLException e) {
                e.printStackTrace();
                outputTextArea.append("Error deleting item from inventory.\n");
            }
        } else {
            outputTextArea.append("You do not have permission to perform this operation.\n");
        }
    }

    private static class User {
        String username;
        String password;
        String role;

        public User(String username, String password, String role) {
            this.username = username;
            this.password = password;
            this.role = role;
        }
    }
}
