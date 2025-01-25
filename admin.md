# Admin Panel

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Panel</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <!-- Admin Section -->
    <div class="admin-section">
        <h1>Admin Panel</h1>

        <?php
        include('db_connection.php'); // Include the database connection

        // Handling the form submission
        if ($_SERVER['REQUEST_METHOD'] == 'POST' && isset($_POST['booking_id']) && isset($_POST['action'])) {
            $bookingId = $_POST['booking_id'];
            $action = $_POST['action'];

            // Set status based on action (accept or cancel)
            $status = ($action == 'accept') ? 'Accepted' : 'Cancelled';

            // Prepared statement to update booking status
            $updateSQL = "UPDATE bookings SET status = ? WHERE booking_id = ?";
            $stmt = mysqli_prepare($conn, $updateSQL);

            if ($stmt === false) {
                die('Error preparing query: ' . mysqli_error($conn));
            }

            mysqli_stmt_bind_param($stmt, 'si', $status, $bookingId);

            if (mysqli_stmt_execute($stmt)) {
                echo "<div class='response'>Booking $status successfully.</div>";

                // Fetch the booking details for SMS notification
                $query = "SELECT * FROM bookings WHERE booking_id = ?";
                $stmt = mysqli_prepare($conn, $query);

                if ($stmt === false) {
                    die('Error preparing fetch query: ' . mysqli_error($conn));
                }

                mysqli_stmt_bind_param($stmt, 'i', $bookingId);
                mysqli_stmt_execute($stmt);
                $result = mysqli_stmt_get_result($stmt);
                $booking = mysqli_fetch_assoc($result);

                // Ensure the phone number is set
                $userPhone = isset($booking['contact_number']) ? $booking['contact_number'] : null;

                if (!$userPhone) {
                    echo "<div class='error'>No phone number available for SMS notification.</div>";
                } else {
                    // Debug: Log the phone number to make sure it's valid
                    echo "<div class='debug'>Phone Number: $userPhone</div>";

                    // Ensure the phone number is in the correct format (E.164 format)
                    if (!preg_match('/^\+?[1-9]\d{1,14}$/', $userPhone)) {
                        echo "<div class='error'>Invalid phone number format. Please ensure it follows the international format (e.g., +1234567890).</div>";
                    } else {
                        // Send SMS notification if the booking is accepted
                        $message = "Hello {$booking['full_name']}, your booking has been {$status}. Here are your booking details:\n";
                        $message .= "Booking Type: {$booking['booking_type']}\n";
                        $message .= "Check-in: {$booking['check_in_date']} {$booking['check_in_time']}\n";
                        $message .= "Check-out: {$booking['check_out_date']} {$booking['check_out_time']}\n";

                        // Infobip API URL and API Key
                        $url = "https://xk24ml.api.infobip.com/sms/2/text/advanced";
                        $apiKey = "75279aaa2c87908cb671a81991072ec8-60c8af0d-8148-4992-ac39-e82a2da76680"; // Use your actual API key

                        // Prepare the data to send to the API
                        $data = [
                            "messages" => [
                                [
                                    "from" => "INFOBIP",
                                    "to" => $userPhone,
                                    "text" => $message
                                ]
                            ]
                        ];
                        
                    }
                }
            } else {
                echo "<div class='error'>Error: " . mysqli_error($conn) . "</div>";
            }

            // Close statement after execution
            mysqli_stmt_close($stmt);
        }

        // Fetch all bookings for the admin to process
        $result = mysqli_query($conn, "SELECT booking_id, full_name, email, booking_type, check_in_date, check_in_time, check_out_date, check_out_time, status, contact_number FROM bookings");

        if (!$result) {
            // Query failed, display error
            die("Error executing query: " . mysqli_error($conn));
        }

        if (mysqli_num_rows($result) > 0) {
            // Display bookings in a table format
            echo "<table><tr><th>ID</th><th>Full Name</th><th>Email</th><th>Phone Number</th><th>Booking Type</th><th>Check-in</th><th>Check-out</th><th>Status</th><th>Action</th></tr>";
            while ($row = mysqli_fetch_assoc($result)) {
                echo "<tr>
                        <td>{$row['booking_id']}</td>
                        <td>{$row['full_name']}</td>
                        <td>{$row['email']}</td>
                        <td>{$row['contact_number']}</td>
                        <td>{$row['booking_type']}</td>
                        <td>{$row['check_in_date']} {$row['check_in_time']}</td>
                        <td>{$row['check_out_date']} {$row['check_out_time']}</td>
                        <td>{$row['status']}</td>
                        <td>
                            <form method='POST'>
                                <input type='hidden' name='booking_id' value='{$row['booking_id']}'>
                                <button type='submit' name='action' value='accept'>Accept</button>
                                <button type='submit' name='action' value='cancel'>Cancel</button>
                            </form>
                        </td>
                    </tr>";
            }
            echo "</table>";
        } else {
            echo "<div class='error'>No available bookings to process.</div>";
        }

        // Close the database connection
        mysqli_close($conn);
        ?>
    </div>
</body>
</html>
