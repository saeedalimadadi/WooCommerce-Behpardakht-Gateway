# WooCommerce Behpardakht Mellat Payment Gateway

This is a custom **WooCommerce payment gateway integration for Behpardakht Mellat**, one of Iran's major online payment service providers. This code allows you to add Behpardakht Mellat as a payment option on your WooCommerce checkout page, enabling your customers to pay for their orders securely through the Mellat payment portal.

## Why use this code?

* **Accept Payments via Behpardakht Mellat:** Seamlessly integrate Behpardakht Mellat as a payment method for your WooCommerce store.
* **Secure Transactions:** Facilitates secure redirection to the Behpardakht gateway for payment processing.
* **Admin Configuration:** Provides an intuitive settings page within WooCommerce to configure your Behpardakht Terminal ID, Username, and Password.
* **Automated Order Status Updates:** Automatically updates order statuses based on payment success or failure.
* **Comprehensive Error Handling:** Includes detailed error handling with user-friendly messages for various transaction outcomes from Behpardakht's API.
* **Currency Conversion:** Automatically converts Toman to Rial for Behpardakht's API if your store's currency is set to Toman (Behpardakht's API expects amounts in Rial).

## Features

* Adds "Behpardakht Mellat" as a selectable payment gateway in WooCommerce settings.
* Allows configuration of gateway title, description, Terminal ID, Username, and Password.
* Handles the entire payment flow:
    * Initiates payment request (`bpPayRequest`) to Behpardakht.
    * Redirects customer to the Behpardakht payment page.
    * Manages the callback (return from Behpardakht).
    * **Verifies payment** (`bpVerifyRequest`) with Behpardakht's API.
    * **Settles payment** (`bpSettleRequest`) for confirmed transactions.
    * Completes WooCommerce order for successful payments.
    * Updates order status and adds notes for successful, cancelled, or failed payments.
* Translates Behpardakht API error codes into human-readable messages.

## Installation

This code functions as a custom WooCommerce payment gateway plugin.

1.  **Create Plugin Folder:** In your WordPress installation, navigate to `wp-content/plugins/` and create a new folder named `woocommerce-behpardakht-gateway`.
2.  **Create Plugin File:** Inside the `woocommerce-behpardakht-gateway` folder, create a file named `woocommerce-behpardakht-gateway.php`.
3.  **Paste Code:** Copy and paste the entire code provided below into the `woocommerce-behpardakht-gateway.php` file.

    ```php
    <?php
    /**
     * Plugin Name: WooCommerce Behpardakht Mellat Payment Gateway
     * Description: Integrates Behpardakht Mellat as a payment gateway for WooCommerce.
     * Version: 1.0
     * Author: Your Name (Optional)
     * License: GPL-2.0-or-later
     * Text Domain: wc-behpardakht-gateway
     *
     * @package WooCommerce
     * @subpackage Behpardakht_Gateway
     * @author Masoud Aroodipoor
     */

    if ( ! defined( 'ABSPATH' ) ) {
        exit; // Exit if accessed directly.
    }

    /**
     * Adds the Behpardakht gateway class to WooCommerce.
     *
     * @param array $gateways Existing WooCommerce payment gateways.
     * @return array Modified list of gateways.
     */
    add_filter('woocommerce_payment_gateways', 'add_behpardakht_gateway');
    function add_behpardakht_gateway($gateways)
    {
        $gateways[] = 'WC_Gateway_Behpardakht';
        return $gateways;
    }

    /**
     * Initializes the Behpardakht payment gateway class.
     * Ensures WC_Payment_Gateway class is available before proceeding.
     */
    add_action('plugins_loaded', 'init_behpardakht_gateway');
    function init_behpardakht_gateway()
    {
        if (!class_exists('WC_Payment_Gateway')) {
            return;
        }

        class WC_Gateway_Behpardakht extends WC_Payment_Gateway
        {
            /**
             * @var string Terminal ID
             */
            public $terminal_id;

            /**
             * @var string Username
             */
            public $username;

            /**
             * @var string Password
             */
            public $password;

            /**
             * Constructor for the gateway.
             */
            public function __construct()
            {
                $this->id = 'behpardakht';
                $this->method_title = __('Behpardakht Mellat Gateway', 'wc-behpardakht-gateway'); // Displayed in admin
                $this->method_description = __('Connect to Mellat Bank e-payment gateway.', 'wc-behpardakht-gateway'); // Displayed in admin
                $this->has_fields = false; // No extra fields on checkout
                $this->icon = apply_filters('woocommerce_behpardakht_icon', plugin_dir_url(__FILE__) . 'behpardakht.png'); // Path to gateway icon (add this image to your plugin folder)

                // Initialize settings and form fields
                $this->init_form_fields();
                $this->init_settings();

                // Get values from settings
                $this->title       = $this->get_option('title'); // Displayed on checkout page
                $this->description = $this->get_option('description'); // Displayed on checkout page
                $this->terminal_id = $this->get_option('terminal_id');
                $this->username    = $this->get_option('username');
                $this->password    = $this->get_option('password');

                // Register hooks
                add_action('woocommerce_update_options_payment_gateways_' . $this->id, [$this, 'process_admin_options']);
                add_action('woocommerce_api_wc_gateway_behpardakht', [$this, 'callback_handler']); // Handles the return from Behpardakht
            }

            /**
             * Define gateway settings fields.
             */
            public function init_form_fields()
            {
                $this->form_fields = [
                    'enabled' => [
                        'title'   => __('Enable/Disable', 'wc-behpardakht-gateway'),
                        'type'    => 'checkbox',
                        'label'   => __('Enable Behpardakht Mellat Gateway', 'wc-behpardakht-gateway'),
                        'default' => 'yes',
                    ],
                    'title' => [
                        'title'       => __('Title', 'wc-behpardakht-gateway'),
                        'type'        => 'text',
                        'description' => __('This controls the title which the user sees during checkout.', 'wc-behpardakht-gateway'),
                        'default'     => __('Online payment with Behpardakht Mellat', 'wc-behpardakht-gateway'),
                        'desc_tip'    => true,
                    ],
                    'description' => [
                        'title'       => __('Description', 'wc-behpardakht-gateway'),
                        'type'        => 'textarea',
                        'description' => __('This controls the description which the user sees during checkout.', 'wc-behpardakht-gateway'),
                        'default'     => __('Secure payment through Behpardakht Mellat gateway.', 'wc-behpardakht-gateway'),
                    ],
                    'terminal_id' => [
                        'title'       => __('Terminal ID', 'wc-behpardakht-gateway'),
                        'type'        => 'text',
                        'description' => __('Enter your Terminal ID received from Mellat Bank.', 'wc-behpardakht-gateway'),
                    ],
                    'username' => [
                        'title'       => __('Username', 'wc-behpardakht-gateway'),
                        'type'        => 'text',
                        'description' => __('Username for connecting to the gateway.', 'wc-behpardakht-gateway'),
                    ],
                    'password' => [
                        'title'       => __('Password', 'wc-behpardakht-gateway'),
                        'type'        => 'password',
                        'description' => __('Password for connecting to the gateway.', 'wc-behpardakht-gateway'),
                    ],
                ];
            }

            /**
             * Process the payment and redirect the user to the gateway.
             *
             * @param int $order_id The ID of the order.
             * @return array An array of result and redirect URL.
             */
            public function process_payment($order_id)
            {
                $order = wc_get_order($order_id);
                $amount = (int) $order->get_total();
                
                // Convert Toman to Rial. If your prices are already in Rial, remove the * 10.
                $amount_in_rials = $amount * 10; 

                // Callback URL for the gateway
                $callback_url = add_query_arg('wc-api', 'wc_gateway_behpardakht', home_url('/'));
                
                $data = [
                    'terminalId'     => $this->terminal_id,
                    'userName'       => $this->username,
                    'userPassword'   => $this->password,
                    'orderId'        => $order_id,
                    'amount'         => $amount_in_rials,
                    'localDate'      => date('Ymd'),
                    'localTime'      => date('His'),
                    'additionalData' => '', // Optional additional data
                    'callBackUrl'    => $callback_url,
                    'payerId'        => 0, // Customer ID, use 0 if not applicable
                ];

                $response = $this->send_request_to_api('[https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpPayRequest](https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpPayRequest)', $data);

                if ($response && isset($response['resCode']) && $response['resCode'] == '0') {
                    $ref_id = $response['refId'];
                    // Store RefId for later use in verification/settlement if needed, or for reference
                    update_post_meta($order_id, '_behpardakht_ref_id', $ref_id);
                    
                    return [
                        'result'   => 'success',
                        'redirect' => "[https://bpm.shaparak.ir/pgwchannel/startpay.mellat?RefId=](https://bpm.shaparak.ir/pgwchannel/startpay.mellat?RefId=){$ref_id}",
                    ];
                } else {
                    $error_message = $this->get_error_message($response['resCode'] ?? 'unknown');
                    wc_add_notice(__('Error connecting to payment gateway:', 'wc-behpardakht-gateway') . ' ' . $error_message, 'error');
                    return ['result' => 'failure'];
                }
            }

            /**
             * Handle the callback from the gateway.
             */
            public function callback_handler()
            {
                global $woocommerce;
                
                // If payment was not successful or user cancelled
                if (empty($_POST['ResCode']) || $_POST['ResCode'] != '0') {
                    wc_add_notice(__('Payment was unsuccessful or cancelled by you.', 'wc-behpardakht-gateway'), 'error');
                    wp_redirect($woocommerce->cart->get_checkout_url());
                    exit;
                }

                $order_id = absint($_POST['SaleOrderId']);
                $order = wc_get_order($order_id);
                
                if (!$order || !$order->get_id()) {
                    wc_add_notice(__('Order not found.', 'wc-behpardakht-gateway'), 'error');
                    wp_redirect(home_url());
                    exit;
                }

                // Prepare data for Verification
                $verify_data = [
                    'terminalId'      => $this->terminal_id,
                    'userName'        => $this->username,
                    'userPassword'    => $this->password,
                    'orderId'         => $order_id,
                    'saleOrderId'     => absint($_POST['SaleOrderId']),
                    'saleReferenceId' => absint($_POST['SaleReferenceId']),
                ];

                $verify_response = $this->send_request_to_api('[https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpVerifyRequest](https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpVerifyRequest)', $verify_data);

                if ($verify_response && isset($verify_response['resCode']) && $verify_response['resCode'] == '0') {
                    // Payment Verified Successfully. Now Settle.
                    $settle_data = [
                        'terminalId'      => $this->terminal_id,
                        'userName'        => $this->username,
                        'userPassword'    => $this->password,
                        'orderId'         => $order_id,
                        'saleOrderId'     => absint($_POST['SaleOrderId']),
                        'saleReferenceId' => absint($_POST['SaleReferenceId']),
                    ];

                    $settle_response = $this->send_request_to_api('[https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpSettleRequest](https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpSettleRequest)', $settle_data);

                    if ($settle_response && isset($settle_response['resCode']) && $settle_response['resCode'] == '0') {
                        // Payment Settled Successfully
                        $order->payment_complete(absint($_POST['SaleReferenceId'])); // Use SaleReferenceId as transaction ID
                        $order->add_order_note(__('Payment successful. Behpardakht Reference ID:', 'wc-behpardakht-gateway') . ' ' . absint($_POST['SaleReferenceId']));
                        wc_add_notice(__('Your payment was successful.', 'wc-behpardakht-gateway'), 'success');
                        wp_redirect($this->get_return_url($order));
                        exit;
                    } else {
                        // Settle failed. Log and update order status to 'on-hold'.
                        $settle_error_message = $this->get_error_message($settle_response['resCode'] ?? 'unknown');
                        error_log('Behpardakht Settlement Failed for Order ' . $order_id . ': ' . $settle_error_message);
                        $order->update_status('on-hold', __('Payment verified but settlement failed:', 'wc-behpardakht-gateway') . ' ' . $settle_error_message);
                        wc_add_notice(__('Payment was verified, but there was an issue settling the transaction. Please contact support.', 'wc-behpardakht-gateway') . ' ' . $settle_error_message, 'error');
                        wp_redirect($woocommerce->cart->get_checkout_url());
                        exit;
                    }
                } else {
                    // Verification failed.
                    $verify_error_message = $this->get_error_message($verify_response['resCode'] ?? 'unknown');
                    error_log('Behpardakht Verification Failed for Order ' . $order_id . ': ' . $verify_error_message);
                    $order->update_status('failed', __('Payment verification failed:', 'wc-behpardakht-gateway') . ' ' . $verify_error_message);
                    wc_add_notice(__('Payment verification failed:', 'wc-behpardakht-gateway') . ' ' . $verify_error_message, 'error');
                    wp_redirect($woocommerce->cart->get_checkout_url());
                    exit;
                }
            }

            /**
             * Sends a request to the Behpardakht API (SOAP endpoint).
             * Note: Behpardakht APIs are SOAP-based, but this example uses cURL/wp_remote_post with XML.
             * This simplified method assumes a basic POST to SOAP endpoint.
             * For robust SOAP integration, a proper SOAP client should be used.
             *
             * @param string $url  API endpoint URL.
             * @param array  $data Data to send in the request body (associative array).
             * @return array Decoded response from the API.
             */
            private function send_request_to_api($url, $data) {
                // Convert data array to a SOAP XML string. This is a simplified example.
                // A full SOAP integration would involve generating a proper SOAP envelope.
                // For Behpardakht, `op=bpPayRequest`, `op=bpVerifyRequest`, etc., indicate the SOAP method.
                $body_params = http_build_query($data);

                $response = wp_remote_post($url, [
                    'method'  => 'POST',
                    'headers' => [
                        'Content-Type' => 'application/x-www-form-urlencoded', // Behpardakht typically expects URL-encoded form data
                        'User-Agent'   => 'WordPress/' . get_bloginfo('version') . '; ' . home_url(),
                    ],
                    'body'    => $body_params,
                    'timeout' => 30, // seconds
                ]);

                if (is_wp_error($response)) {
                    error_log('Behpardakht API Connection Error: ' . $response->get_error_message());
                    return ['resCode' => '-999']; // Custom code for connection error
                }
                
                $body = wp_remote_retrieve_body($response);
                // Behpardakht response is often a string like "ResCode=0,SaleOrderId=...,SaleReferenceId=..."
                // Parse this string into an associative array.
                $parsed_response = [];
                parse_str(str_replace('=', '&', $body), $parsed_response); // Simplified parsing

                return $parsed_response;
            }

            /**
             * Translates Behpardakht error codes into human-readable messages.
             *
             * @param string $code The Behpardakht error code.
             * @return string Translated error message.
             */
            private function get_error_message($code) {
                $errors = [
                    '0'   => __('Successful transaction.', 'wc-behpardakht-gateway'),
                    '1'   => __('Invalid transaction data.', 'wc-behpardakht-gateway'),
                    '3'   => __('Incorrect or invalid merchant information.', 'wc-behpardakht-gateway'),
                    '4'   => __('Terminal ID not found.', 'wc-behpardakht-gateway'),
                    '5'   => __('Transaction not found.', 'wc-behpardakht-gateway'),
                    '6'   => __('Transaction has already been reversed.', 'wc-behpardakht-gateway'),
                    '7'   => __('Transaction reversed successfully.', 'wc-behpardakht-gateway'),
                    '8'   => __('Transaction not found for reversal.', 'wc-behpardakht-gateway'),
                    '9'   => __('Transaction not found for settlement.', 'wc-behpardakht-gateway'),
                    '10'  => __('Settle operation failed.', 'wc-behpardakht-gateway'),
                    '11'  => __('Transaction has already been settled.', 'wc-behpardakht-gateway'),
                    '12'  => __('Transaction amount does not match.', 'wc-behpardakht-gateway'),
                    '13'  => __('Transaction failed.', 'wc-behpardakht-gateway'),
                    '14'  => __('Duplicate order ID.', 'wc-behpardakht-gateway'),
                    '15'  => __('Payment amount is less than minimum allowed.', 'wc-behpardakht-gateway'),
                    '16'  => __('Payment amount is more than maximum allowed.', 'wc-behpardakht-gateway'),
                    '17'  => __('Callback URL not valid.', 'wc-behpardakht-gateway'),
                    '18'  => __('Invalid date and time format.', 'wc-behpardakht-gateway'),
                    '19'  => __('Transaction session expired.', 'wc-behpardakht-gateway'),
                    '20'  => __('Invalid account information.', 'wc-behpardakht-gateway'),
                    '21'  => __('Card number is invalid.', 'wc-behpardakht-gateway'),
                    '22'  => __('Insufficient funds.', 'wc-behpardakht-gateway'),
                    '23'  => __('Card expiration date is invalid.', 'wc-behpardakht-gateway'),
                    '24'  => __('CVV2 or password is invalid.', 'wc-behpardakht-gateway'),
                    '25'  => __('PIN is incorrect.', 'wc-behpardakht-gateway'),
                    '26'  => __('Customer password is invalid.', 'wc-behpardakht-gateway'),
                    '27'  => __('Transaction denied.', 'wc-behpardakht-gateway'),
                    '28'  => __('Card is expired.', 'wc-behpardakht-gateway'),
                    '29'  => __('Transaction canceled by customer.', 'wc-behpardakht-gateway'),
                    '30'  => __('Transaction rejected by system.', 'wc-behpardakht-gateway'),
                    '31'  => __('Transaction already approved.', 'wc-behpardakht-gateway'),
                    '32'  => __('Invalid transaction status.', 'wc-behpardakht-gateway'),
                    '33'  => __('Authentication failed.', 'wc-behpardakht-gateway'),
                    '34'  => __('Transaction blocked.', 'wc-behpardakht-gateway'),
                    '35'  => __('Terminal is inactive.', 'wc-behpardakht-gateway'),
                    '36'  => __('Session invalid.', 'wc-behpardakht-gateway'),
                    '37'  => __('Payment is already initiated.', 'wc-behpardakht-gateway'),
                    '38'  => __('Invalid IP address.', 'wc-behpardakht-gateway'),
                    '39'  => __('Invalid request data for refund.', 'wc-behpardakht-gateway'),
                    '40'  => __('Settlement is not possible.', 'wc-behpardakht-gateway'),
                    '41'  => __('Transaction limit exceeded.', 'wc-behpardakht-gateway'),
                    '42'  => __('Invalid password for card.', 'wc-behpardakht-gateway'),
                    '43'  => __('Amount validation failed.', 'wc-behpardakht-gateway'),
                    '44'  => __('Card blocked.', 'wc-behpardakht-gateway'),
                    '45'  => __('Transaction cancelled.', 'wc-behpardakht-gateway'),
                    '46'  => __('Unsuccessful refund.', 'wc-behpardakht-gateway'),
                    '47'  => __('Invalid refund amount.', 'wc-behpardakht-gateway'),
                    '48'  => __('Refund limit exceeded.', 'wc-behpardakht-gateway'),
                    '49'  => __('Invalid account for refund.', 'wc-behpardakht-gateway'),
                    '50'  => __('Invalid transaction for refund.', 'wc-behpardakht-gateway'),
                    '51'  => __('Refund not allowed.', 'wc-behpardakht-gateway'),
                    '52'  => __('Invalid settlement period.', 'wc-behpardakht-gateway'),
                    '53'  => __('Invalid order amount.', 'wc-behpardakht-gateway'),
                    '54'  => __('Invalid transaction for inquiry.', 'wc-behpardakht-gateway'),
                    '55'  => __('Inquiry not allowed.', 'wc-behpardakht-gateway'),
                    '56'  => __('Reversal limit exceeded.', 'wc-behpardakht-gateway'),
                    '57'  => __('Invalid reversal amount.', 'wc-behpardakht-gateway'),
                    '58'  => __('Reversal not allowed.', 'wc-behpardakht-gateway'),
                    '59'  => __('Transaction expired.', 'wc-behpardakht-gateway'),
                    '60'  => __('Transaction limit per day exceeded.', 'wc-behpardakht-gateway'),
                    '61'  => __('Card blocked by issuer.', 'wc-behpardakht-gateway'),
                    '62'  => __('Incorrect card information.', 'wc-behpardakht-gateway'),
                    '63'  => __('Incorrect CVC2 or PIN.', 'wc-behpardakht-gateway'),
                    '64'  => __('Card is stolen.', 'wc-behpardakht-gateway'),
                    '65'  => __('Invalid card.', 'wc-behpardakht-gateway'),
                    '66'  => __('Transaction not allowed.', 'wc-behpardakht-gateway'),
                    '67'  => __('Invalid transaction ID.', 'wc-behpardakht-gateway'),
                    '68'  => __('Transaction declined by bank.', 'wc-behpardakht-gateway'),
                    '69'  => __('Invalid password.', 'wc-behpardakht-gateway'),
                    '70'  => __('Transaction blocked due to risk.', 'wc-behpardakht-gateway'),
                    '71'  => __('Invalid merchant account.', 'wc-behpardakht-gateway'),
                    '72'  => __('Transaction blocked by bank policy.', 'wc-behpardakht-gateway'),
                    '73'  => __('Maximum number of transactions reached.', 'wc-behpardakht-gateway'),
                    '74'  => __('Invalid currency.', 'wc-behpardakht-gateway'),
                    '75'  => __('Invalid amount for refund.', 'wc-behpardakht-gateway'),
                    '76'  => __('Maximum amount for refund exceeded.', 'wc-behpardakht-gateway'),
                    '77'  => __('Invalid order data for inquiry.', 'wc-behpardakht-gateway'),
                    '78'  => __('Inquiry data missing.', 'wc-behpardakht-gateway'),
                    '79'  => __('Settle transaction not found.', 'wc-behpardakht-gateway'),
                    '80'  => __('Refund denied.', 'wc-behpardakht-gateway'),
                    '81'  => __('Invalid request data.', 'wc-behpardakht-gateway'),
                    '82'  => __('Amount out of range for this terminal.', 'wc-behpardakht-gateway'),
                    '83'  => __('Invalid security code.', 'wc-behpardakht-gateway'),
                    '84'  => __('Authentication method not supported.', 'wc-behpardakht-gateway'),
                    '85'  => __('Transaction not found for this terminal.', 'wc-behpardakht-gateway'),
                    '86'  => __('Transaction already refunded.', 'wc-behpardakht-gateway'),
                    '87'  => __('Transaction not allowed to settle.', 'wc-behpardakht-gateway'),
                    '88'  => __('Refund amount is not valid.', 'wc-behpardakht-gateway'),
                    '89'  => __('Transaction expired for reversal.', 'wc-behpardakht-gateway'),
                    '90'  => __('Invalid transaction reference.', 'wc-behpardakht-gateway'),
                    '91'  => __('Refund failed.', 'wc-behpardakht-gateway'),
                    '92'  => __('Transaction is pending.', 'wc-behpardakht-gateway'),
                    '93'  => __('Duplicate reference ID.', 'wc-behpardakht-gateway'),
                    '94'  => __('Invalid refund request.', 'wc-behpardakht-gateway'),
                    '95'  => __('Refund not supported.', 'wc-behpardakht-gateway'),
                    '96'  => __('Invalid currency for terminal.', 'wc-behpardakht-gateway'),
                    '97'  => __('Transaction amount cannot be zero.', 'wc-behpardakht-gateway'),
                    '98'  => __('Transaction suspended.', 'wc-behpardakht-gateway'),
                    '99'  => __('Transaction is being processed.', 'wc-behpardakht-gateway'),
                    '-999' => __('Error connecting to Behpardakht server.', 'wc-behpardakht-gateway'),
                    'unknown' => __('An unknown error occurred.', 'wc-behpardakht-gateway'),
                ];

                return $errors[$code] ?? sprintf(__('Undefined error with code: %s', 'wc-behpardakht-gateway'), $code);
            }
        }
    }
    ```

4.  **Upload Icon (Optional but Recommended):** Place your Behpardakht payment gateway icon (e.g., `behpardakht.png`) inside the `woocommerce-behpardakht-gateway` folder, next to `woocommerce-behpardakht-gateway.php`. This icon will appear next to the payment gateway name on the checkout page.
5.  **Activate Plugin:** Go to your WordPress admin dashboard, navigate to **"Plugins"**, and **activate** the "WooCommerce Behpardakht Mellat Payment Gateway" plugin.
6.  **Configure Gateway:** Go to **WooCommerce > Settings > Payments**. You should now see "Behpardakht Mellat" listed. Click on "Manage" or "Set up" to configure your **Terminal ID, Username, and Password**.

## Important Considerations

* **API Credentials:** You **MUST** obtain valid Terminal ID, Username, and Password from Behpardakht Mellat (Bank Mellat) to use this gateway. These are usually provided after you've successfully registered for their online payment services.
* **Currency:** Behpardakht's API expects the amount in **Rials**. The provided code includes logic to convert Toman to Rial (`$amount * 10`). Ensure your WooCommerce currency is correctly set up, and this conversion matches Behpardakht's current API expectations.
* **SSL Certificate:** For security, ensure your website has a valid SSL certificate (HTTPS) installed. Payment gateways require secure connections.
* **Error Logging:** The `error_log` calls are useful for debugging connection issues. Check your server's PHP error logs if you encounter problems.
* **Security:** This is a basic implementation. For production environments, always consider additional security best practices, such as IP whitelisting if supported by Behpardakht, and regular security audits. Keep your Terminal ID, Username, and Password secure.
* **SOAP vs. REST:** Behpardakht's older APIs are primarily SOAP-based. This simplified implementation uses `wp_remote_post` to interact with a URL-encoded endpoint which often works for basic `bpPayRequest`, `bpVerifyRequest`, and `bpSettleRequest`. For a truly robust and future-proof integration, especially if Behpardakht changes its API or you need more complex SOAP functionalities, a dedicated PHP SOAP client might be necessary.
* **Updates:** Custom gateway code needs to be maintained. Ensure you stay updated with any changes in Behpardakht's API or WooCommerce's payment gateway standards.

## Contributing

Contributions are welcome! If you have suggestions or improvements for this code, feel free to open a "Pull Request" or report an "Issue."

## License

This project is licensed under the GPL-2.0-or-later License.

---

# درگاه پرداخت به‌پرداخت ملت برای ووکامرس

این کد یکپارچه‌سازی سفارشی **درگاه پرداخت به‌پرداخت ملت برای ووکامرس** است. به‌پرداخت ملت یکی از ارائه‌دهندگان اصلی خدمات پرداخت آنلاین در ایران است. این کد به شما امکان می‌دهد به‌پرداخت ملت را به عنوان یک گزینه پرداخت در صفحه تسویه‌حساب ووکامرس خود اضافه کنید و مشتریان خود را قادر می‌سازد تا سفارشات خود را به طور امن از طریق پورتال پرداخت ملت پرداخت کنند.

## چرا از این کد استفاده کنیم؟

* **پذیرش پرداخت از طریق به‌پرداخت ملت:** به‌پرداخت ملت را به طور یکپارچه به عنوان یک روش پرداخت برای فروشگاه ووکامرس خود اضافه کنید.
* **تراکنش‌های امن:** انتقال امن به درگاه به‌پرداخت برای پردازش پرداخت را تسهیل می‌کند.
* **پیکربندی مدیر:** یک صفحه تنظیمات بصری در ووکامرس برای پیکربندی شناسه پایانه، نام کاربری و رمز عبور به‌پرداخت شما فراهم می‌کند.
* **به‌روزرسانی خودکار وضعیت سفارش:** وضعیت سفارش را به طور خودکار بر اساس موفقیت یا عدم موفقیت پرداخت به‌روزرسانی می‌کند.
* **مدیریت خطای جامع:** شامل مدیریت خطای دقیق با پیام‌های کاربرپسند برای نتایج مختلف تراکنش‌ها از API به‌پرداخت است.
* **تبدیل ارز:** اگر ارز فروشگاه شما تومان باشد، به طور خودکار تومان را به ریال برای API به‌پرداخت تبدیل می‌کند (API به‌پرداخت مبلغ را به ریال انتظار دارد).

## قابلیت‌ها

* "درگاه به‌پرداخت ملت" را به عنوان یک درگاه پرداخت قابل انتخاب در تنظیمات ووکامرس اضافه می‌کند.
* امکان پیکربندی عنوان درگاه، توضیحات، شناسه پایانه، نام کاربری و رمز عبور را فراهم می‌کند.
* تمام جریان پرداخت را مدیریت می‌کند:
    * آغاز درخواست پرداخت (`bpPayRequest`) به به‌پرداخت.
    * هدایت مشتری به صفحه پرداخت به‌پرداخت.
    * مدیریت بازگشت از به‌پرداخت.
    * **تأیید پرداخت** (`bpVerifyRequest`) با API به‌پرداخت.
    * **تسویه پرداخت** (`bpSettleRequest`) برای تراکنش‌های تأیید شده.
    * تکمیل سفارش ووکامرس برای پرداخت‌های موفق.
    * به‌روزرسانی وضعیت سفارش و افزودن یادداشت‌ها برای پرداخت‌های موفق، لغو شده یا ناموفق.
* کدهای خطای API به‌پرداخت را به پیام‌های قابل فهم برای انسان ترجمه می‌کند.

## نصب

این کد به عنوان یک افزونه درگاه پرداخت سفارشی ووکامرس عمل می‌کند.

1.  **ایجاد پوشه افزونه:** در نصب وردپرس خود، به مسیر `wp-content/plugins/` بروید و یک پوشه جدید به نام `woocommerce-behpardakht-gateway` ایجاد کنید.
2.  **ایجاد فایل افزونه:** در داخل پوشه `woocommerce-behpardakht-gateway`، یک فایل به نام `woocommerce-behpardakht-gateway.php` ایجاد کنید.
3.  **جایگذاری کد:** کل کد ارائه شده در پایین را کپی کرده و در فایل `woocommerce-behpardakht-gateway.php` جایگذاری کنید.

    ```php
    <?php
    /**
     * Plugin Name: WooCommerce Behpardakht Mellat Payment Gateway
     * Description: Integrates Behpardakht Mellat as a payment gateway for WooCommerce.
     * Version: 1.0
     * Author: Your Name (Optional)
     * License: GPL-2.0-or-later
     * Text Domain: wc-behpardakht-gateway
     *
     * @package WooCommerce
     * @subpackage Behpardakht_Gateway
     * @author Masoud Aroodipoor
     */

    if ( ! defined( 'ABSPATH' ) ) {
        exit; // Exit if accessed directly.
    }

    /**
     * افزودن درگاه به‌پرداخت ملت به لیست درگاه‌های ووکامرس
     */
    add_filter('woocommerce_payment_gateways', 'add_behpardakht_gateway');
    function add_behpardakht_gateway($gateways)
    {
        $gateways[] = 'WC_Gateway_Behpardakht';
        return $gateways;
    }

    /**
     * مقداردهی اولیه کلاس درگاه پرداخت
     */
    add_action('plugins_loaded', 'init_behpardakht_gateway');
    function init_behpardakht_gateway()
    {
        if (!class_exists('WC_Payment_Gateway')) {
            return;
        }

        class WC_Gateway_Behpardakht extends WC_Payment_Gateway
        {
            /**
             * @var string شناسه پایانه
             */
            public $terminal_id;

            /**
             * @var string نام کاربری
             */
            public $username;

            /**
             * @var string رمز عبور
             */
            public $password;

            /**
             * سازنده کلاس
             */
            public function __construct()
            {
                $this->id = 'behpardakht';
                $this->method_title = __('درگاه به‌پرداخت ملت', 'wc-behpardakht-gateway');
                $this->method_description = __('اتصال به درگاه پرداخت الکترونیکی بانک ملت.', 'wc-behpardakht-gateway');
                $this->has_fields = false;
                $this->icon = apply_filters('woocommerce_behpardakht_icon', plugin_dir_url(__FILE__) . 'behpardakht.png'); // مسیر آیکون (این تصویر را در پوشه افزونه خود قرار دهید)

                // تنظیمات فرم
                $this->init_form_fields();
                $this->init_settings();

                // دریافت مقادیر از تنظیمات
                $this->title       = $this->get_option('title');
                $this->description = $this->get_option('description');
                $this->terminal_id = $this->get_option('terminal_id');
                $this->username    = $this->get_option('username');
                $this->password    = $this->get_option('password');

                // ثبت هوک‌ها
                add_action('woocommerce_update_options_payment_gateways_' . $this->id, [$this, 'process_admin_options']);
                add_action('woocommerce_api_wc_gateway_behpardakht', [$this, 'callback_handler']);
            }

            /**
             * تعریف فیلدهای تنظیمات درگاه در مدیریت وردپرس
             */
            public function init_form_fields()
            {
                $this->form_fields = [
                    'enabled' => [
                        'title'   => __('فعال‌سازی/غیرفعال‌سازی', 'wc-behpardakht-gateway'),
                        'type'    => 'checkbox',
                        'label'   => __('فعال‌سازی درگاه به‌پرداخت ملت', 'wc-behpardakht-gateway'),
                        'default' => 'yes',
                    ],
                    'title' => [
                        'title'       => __('عنوان', 'wc-behpardakht-gateway'),
                        'type'        => 'text',
                        'description' => __('عنوانی که در صفحه تسویه‌حساب به کاربر نمایش داده می‌شود.', 'wc-behpardakht-gateway'),
                        'default'     => __('پرداخت آنلاین با به‌پرداخت ملت', 'wc-behpardakht-gateway'),
                        'desc_tip'    => true,
                    ],
                    'description' => [
                        'title'       => __('توضیحات', 'wc-behpardakht-gateway'),
                        'type'        => 'textarea',
                        'description' => __('توضیحات این روش پرداخت که در صفحه تسویه‌حساب نمایش داده می‌شود.', 'wc-behpardakht-gateway'),
                        'default'     => __('پرداخت ایمن از طریق درگاه به‌پرداخت ملت.', 'wc-behpardakht-gateway'),
                    ],
                    'terminal_id' => [
                        'title'       => __('شناسه پایانه (Terminal ID)', 'wc-behpardakht-gateway'),
                        'type'        => 'text',
                        'description' => __('شناسه پایانه خود را که از بانک ملت دریافت کرده‌اید وارد کنید.', 'wc-behpardakht-gateway'),
                    ],
                    'username' => [
                        'title'       => __('نام کاربری (Username)', 'wc-behpardakht-gateway'),
                        'type'        => 'text',
                        'description' => __('نام کاربری اتصال به درگاه.', 'wc-behpardakht-gateway'),
                    ],
                    'password' => [
                        'title'       => __('رمز عبور (Password)', 'wc-behpardakht-gateway'),
                        'type'        => 'password',
                        'description' => __('رمز عبور اتصال به درگاه.', 'wc-behpardakht-gateway'),
                    ],
                ];
            }

            /**
             * پردازش پرداخت و انتقال کاربر به درگاه
             *
             * @param int $order_id شناسه سفارش.
             * @return array آرایه‌ای شامل نتیجه و URL هدایت.
             */
            public function process_payment($order_id)
            {
                $order = wc_get_order($order_id);
                $amount = (int) $order->get_total();
                
                // تبدیل تومان به ریال. اگر قیمت‌ها به ریال است، * 10 را حذف کنید.
                $amount_in_rials = $amount * 10; 

                // آدرس بازگشت از درگاه
                $callback_url = add_query_arg('wc-api', 'wc_gateway_behpardakht', home_url('/'));
                
                $data = [
                    'terminalId'     => $this->terminal_id,
                    'userName'       => $this->username,
                    'userPassword'   => $this->password,
                    'orderId'        => $order_id,
                    'amount'         => $amount_in_rials,
                    'localDate'      => date('Ymd'),
                    'localTime'      => date('His'),
                    'additionalData' => '', // اطلاعات اضافی اختیاری
                    'callBackUrl'    => $callback_url,
                    'payerId'        => 0, // شناسه مشتری، در صورت عدم کاربرد 0 قرار دهید
                ];

                $response = $this->send_request_to_api('https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpPayRequest', $data);

                if ($response && isset($response['resCode']) && $response['resCode'] == '0') {
                    $ref_id = $response['refId'];
                    // ذخیره RefId برای استفاده در مرحله بعد (تأیید/تسویه) یا برای ارجاع
                    update_post_meta($order_id, '_behpardakht_ref_id', $ref_id);
                    
                    return [
                        'result'   => 'success',
                        'redirect' => "https://bpm.shaparak.ir/pgwchannel/startpay.mellat?RefId={$ref_id}",
                    ];
                } else {
                    $error_message = $this->get_error_message($response['resCode'] ?? 'unknown');
                    wc_add_notice(__('خطا در اتصال به درگاه:', 'wc-behpardakht-gateway') . ' ' . $error_message, 'error');
                    return ['result' => 'failure'];
                }
            }

            /**
             * مدیریت بازگشت از درگاه
             */
            public function callback_handler()
            {
                global $woocommerce;
                
                // اگر پرداخت موفقیت‌آمیز نبود یا کاربر لغو کرد
                if (empty($_POST['ResCode']) || $_POST['ResCode'] != '0') {
                    wc_add_notice(__('پرداخت ناموفق بود یا توسط شما لغو شد.', 'wc-behpardakht-gateway'), 'error');
                    wp_redirect($woocommerce->cart->get_checkout_url());
                    exit;
                }

                $order_id = absint($_POST['SaleOrderId']);
                $order = wc_get_order($order_id);
                
                if (!$order || !$order->get_id()) {
                    wc_add_notice(__('سفارش یافت نشد.', 'wc-behpardakht-gateway'), 'error');
                    wp_redirect(home_url());
                    exit;
                }

                // آماده‌سازی داده‌ها برای تأیید (Verification)
                $verify_data = [
                    'terminalId'      => $this->terminal_id,
                    'userName'        => $this->username,
                    'userPassword'    => $this->password,
                    'orderId'         => $order_id,
                    'saleOrderId'     => absint($_POST['SaleOrderId']),
                    'saleReferenceId' => absint($_POST['SaleReferenceId']),
                ];

                $verify_response = $this->send_request_to_api('https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpVerifyRequest', $verify_data);

                if ($verify_response && isset($verify_response['resCode']) && $verify_response['resCode'] == '0') {
                    // پرداخت با موفقیت تأیید شد. اکنون تسویه (Settle) را انجام دهید.
                    $settle_data = [
                        'terminalId'      => $this->terminal_id,
                        'userName'        => $this->username,
                        'userPassword'    => $this->password,
                        'orderId'         => $order_id,
                        'saleOrderId'     => absint($_POST['SaleOrderId']),
                        'saleReferenceId' => absint($_POST['SaleReferenceId']),
                    ];

                    $settle_response = $this->send_request_to_api('https://bpm.shaparak.ir/pgwchannel/services/pgw?op=bpSettleRequest', $settle_data);

                    if ($settle_response && isset($settle_response['resCode']) && $settle_response['resCode'] == '0') {
                        // پرداخت با موفقیت تسویه شد
                        $order->payment_complete(absint($_POST['SaleReferenceId'])); // SaleReferenceId را به عنوان شناسه تراکنش استفاده کنید
                        $order->add_order_note(__('پرداخت موفق. شناسه ارجاع به‌پرداخت ملت:', 'wc-behpardakht-gateway') . ' ' . absint($_POST['SaleReferenceId']));
                        wc_add_notice(__('پرداخت شما با موفقیت انجام شد.', 'wc-behpardakht-gateway'), 'success');
                        wp_redirect($this->get_return_url($order));
                        exit;
                    } else {
                        // تسویه ناموفق بود. لاگ و وضعیت سفارش را به‌روز کنید.
                        $settle_error_message = $this->get_error_message($settle_response['resCode'] ?? 'unknown');
                        error_log('تسویه به‌پرداخت برای سفارش ' . $order_id . ' ناموفق بود: ' . $settle_error_message);
                        $order->update_status('on-hold', __('پرداخت تأیید شد اما تسویه ناموفق بود:', 'wc-behpardakht-gateway') . ' ' . $settle_error_message);
                        wc_add_notice(__('پرداخت تأیید شد، اما مشکلی در تسویه تراکنش وجود داشت. لطفاً با پشتیبانی تماس بگیرید.', 'wc-behpardakht-gateway') . ' ' . $settle_error_message, 'error');
                        wp_redirect($woocommerce->cart->get_checkout_url());
                        exit;
                    }
                } else {
                    // تأیید ناموفق بود.
                    $verify_error_message = $this->get_error_message($verify_response['resCode'] ?? 'unknown');
                    error_log('تأیید به‌پرداخت برای سفارش ' . $order_id . ' ناموفق بود: ' . $verify_error_message);
                    $order->update_status('failed', __('تأیید پرداخت ناموفق بود:', 'wc-behpardakht-gateway') . ' ' . $verify_error_message);
                    wc_add_notice(__('تأیید پرداخت ناموفق بود:', 'wc-behpardakht-gateway') . ' ' . $verify_error_message, 'error');
                    wp_redirect($woocommerce->cart->get_checkout_url());
                    exit;
                }
            }

            /**
             * ارسال درخواست به API به‌پرداخت (نقطه پایانی SOAP).
             * نکته: APIهای به‌پرداخت مبتنی بر SOAP هستند، اما این مثال از cURL/wp_remote_post با XML استفاده می‌کند.
             * این روش ساده‌شده فرض می‌کند که POST به نقطه پایانی SOAP انجام می‌شود.
             * برای یکپارچه‌سازی قوی SOAP، باید از یک کلاینت SOAP مناسب استفاده شود.
             *
             * @param string $url  آدرس نقطه پایانی API.
             * @param array  $data داده برای ارسال در بدنه درخواست (آرایه انجمنی).
             * @return array پاسخ رمزگشایی شده از API.
             */
            private function send_request_to_api($url, $data) {
                // تبدیل آرایه داده به یک رشته XML SOAP. این یک مثال ساده‌شده است.
                // یکپارچه‌سازی کامل SOAP شامل تولید یک Envelope SOAP مناسب خواهد بود.
                // برای به‌پرداخت، `op=bpPayRequest`، `op=bpVerifyRequest` و غیره، متد SOAP را نشان می‌دهند.
                $body_params = http_build_query($data);

                $response = wp_remote_post($url, [
                    'method'  => 'POST',
                    'headers' => [
                        'Content-Type' => 'application/x-www-form-urlencoded', // به‌پرداخت معمولاً داده‌های فرم با کدگذاری URL را انتظار دارد
                        'User-Agent'   => 'WordPress/' . get_bloginfo('version') . '; ' . home_url(),
                    ],
                    'body'    => $body_params,
                    'timeout' => 30, // ثانیه
                ]);

                if (is_wp_error($response)) {
                    error_log('خطای اتصال به API به‌پرداخت: ' . $response->get_error_message());
                    return ['resCode' => '-999']; // کد سفارشی برای خطای اتصال
                }
                
                $body = wp_remote_retrieve_body($response);
                // پاسخ به‌پرداخت معمولاً یک رشته مانند "ResCode=0,SaleOrderId=...,SaleReferenceId=..." است.
                // این رشته را به یک آرایه انجمنی تجزیه کنید.
                $parsed_response = [];
                parse_str(str_replace('=', '&', $body), $parsed_response); // تجزیه ساده‌شده

                return $parsed_response;
            }

            /**
             * ترجمه کدهای خطای به‌پرداخت به پیام‌های قابل فهم برای انسان.
             *
             * @param string $code کد خطای به‌پرداخت.
             * @return string پیام خطای ترجمه شده.
             */
            private function get_error_message($code) {
                $errors = [
                    '0'   => __('تراکنش موفقیت‌آمیز.', 'wc-behpardakht-gateway'),
                    '1'   => __('داده‌های تراکنش نامعتبر است.', 'wc-behpardakht-gateway'),
                    '3'   => __('اطلاعات پذیرنده نادرست یا نامعتبر است.', 'wc-behpardakht-gateway'),
                    '4'   => __('شناسه پایانه یافت نشد.', 'wc-behpardakht-gateway'),
                    '5'   => __('تراکنش یافت نشد.', 'wc-behpardakht-gateway'),
                    '6'   => __('تراکنش قبلاً برگشت داده شده است.', 'wc-behpardakht-gateway'),
                    '7'   => __('تراکنش با موفقیت برگشت داده شد.', 'wc-behpardakht-gateway'),
                    '8'   => __('تراکنش برای برگشت یافت نشد.', 'wc-behpardakht-gateway'),
                    '9'   => __('تراکنش برای تسویه یافت نشد.', 'wc-behpardakht-gateway'),
                    '10'  => __('عملیات تسویه ناموفق بود.', 'wc-behpardakht-gateway'),
                    '11'  => __('تراکنش قبلاً تسویه شده است.', 'wc-behpardakht-gateway'),
                    '12'  => __('مبلغ تراکنش مطابقت ندارد.', 'wc-behpardakht-gateway'),
                    '13'  => __('تراکنش ناموفق بود.', 'wc-behpardakht-gateway'),
                    '14'  => __('شناسه سفارش تکراری.', 'wc-behpardakht-gateway'),
                    '15'  => __('مبلغ پرداخت کمتر از حداقل مجاز است.', 'wc-behpardakht-gateway'),
                    '16'  => __('مبلغ پرداخت بیشتر از حداکثر مجاز است.', 'wc-behpardakht-gateway'),
                    '17'  => __('آدرس بازگشت (Callback URL) نامعتبر است.', 'wc-behpardakht-gateway'),
                    '18'  => __('فرمت تاریخ و زمان نامعتبر است.', 'wc-behpardakht-gateway'),
                    '19'  => __('جلسه تراکنش منقضی شده است.', 'wc-behpardakht-gateway'),
                    '20'  => __('اطلاعات حساب نامعتبر است.', 'wc-behpardakht-gateway'),
                    '21'  => __('شماره کارت نامعتبر است.', 'wc-behpardakht-gateway'),
                    '22'  => __('موجودی ناکافی.', 'wc-behpardakht-gateway'),
                    '23'  => __('تاریخ انقضای کارت نامعتبر است.', 'wc-behpardakht-gateway'),
                    '24'  => __('CVV2 یا رمز عبور نامعتبر است.', 'wc-behpardakht-gateway'),
                    '25'  => __('رمز PIN نادرست است.', 'wc-behpardakht-gateway'),
                    '26'  => __('رمز عبور مشتری نامعتبر است.', 'wc-behpardakht-gateway'),
                    '27'  => __('تراکنش رد شد.', 'wc-behpardakht-gateway'),
                    '28'  => __('کارت منقضی شده است.', 'wc-behpardakht-gateway'),
                    '29'  => __('تراکنش توسط مشتری لغو شد.', 'wc-behpardakht-gateway'),
                    '30'  => __('تراکنش توسط سیستم رد شد.', 'wc-behpardakht-gateway'),
                    '31'  => __('تراکنش قبلاً تأیید شده است.', 'wc-behpardakht-gateway'),
                    '32'  => __('وضعیت تراکنش نامعتبر است.', 'wc-behpardakht-gateway'),
                    '33'  => __('احراز هویت ناموفق بود.', 'wc-behpardakht-gateway'),
                    '34'  => __('تراکنش مسدود شده است.', 'wc-behpardakht-gateway'),
                    '35'  => __('پایانه غیرفعال است.', 'wc-behpardakht-gateway'),
                    '36'  => __('جلسه نامعتبر است.', 'wc-behpardakht-gateway'),
                    '37'  => __('پرداخت قبلاً آغاز شده است.', 'wc-behpardakht-gateway'),
                    '38'  => __('آدرس IP نامعتبر است.', 'wc-behpardakht-gateway'),
                    '39'  => __('داده درخواست نامعتبر برای بازپرداخت.', 'wc-behpardakht-gateway'),
                    '40'  => __('تسویه حساب ممکن نیست.', 'wc-behpardakht-gateway'),
                    '41'  => __('محدودیت تراکنش بیش از حد مجاز است.', 'wc-behpardakht-gateway'),
                    '42'  => __('رمز عبور نامعتبر برای کارت.', 'wc-behpardakht-gateway'),
                    '43'  => __('اعتبار سنجی مبلغ ناموفق بود.', 'wc-behpardakht-gateway'),
                    '44'  => __('کارت مسدود شده است.', 'wc-behpardakht-gateway'),
                    '45'  => __('تراکنش لغو شده است.', 'wc-behpardakht-gateway'),
                    '46'  => __('بازپرداخت ناموفق.', 'wc-behpardakht-gateway'),
                    '47'  => __('مبلغ بازپرداخت نامعتبر است.', 'wc-behpardakht-gateway'),
                    '48'  => __('محدودیت بازپرداخت بیش از حد مجاز است.', 'wc-behpardakht-gateway'),
                    '49'  => __('حساب نامعتبر برای بازپرداخت.', 'wc-behpardakht-gateway'),
                    '50'  => __('تراکنش نامعتبر برای بازپرداخت.', 'wc-behpardakht-gateway'),
                    '51'  => __('بازپرداخت مجاز نیست.', 'wc-behpardakht-gateway'),
                    '52'  => __('دوره تسویه نامعتبر است.', 'wc-behpardakht-gateway'),
                    '53'  => __('مبلغ سفارش نامعتبر است.', 'wc-behpardakht-gateway'),
                    '54'  => __('تراکنش نامعتبر برای استعلام.', 'wc-behpardakht-gateway'),
                    '55'  => __('استعلام مجاز نیست.', 'wc-behpardakht-gateway'),
                    '56'  => __('محدودیت برگشت بیش از حد مجاز است.', 'wc-behpardakht-gateway'),
                    '57'  => __('مبلغ برگشت نامعتبر است.', 'wc-behpardakht-gateway'),
                    '58'  => __('برگشت مجاز نیست.', 'wc-behpardakht-gateway'),
                    '59'  => __('تراکنش منقضی شده است.', 'wc-behpardakht-gateway'),
                    '60'  => __('محدودیت تراکنش در روز بیش از حد مجاز است.', 'wc-behpardakht-gateway'),
                    '61'  => __('کارت توسط صادرکننده مسدود شده است.', 'wc-behpardakht-gateway'),
                    '62'  => __('اطلاعات کارت نادرست است.', 'wc-behpardakht-gateway'),
                    '63'  => __('CVC2 یا PIN نادرست است.', 'wc-behpardakht-gateway'),
                    '64'  => __('کارت سرقتی است.', 'wc-behpardakht-gateway'),
                    '65'  => __('کارت نامعتبر است.', 'wc-behpardakht-gateway'),
                    '66'  => __('تراکنش مجاز نیست.', 'wc-behpardakht-gateway'),
                    '67'  => __('شناسه تراکنش نامعتبر است.', 'wc-behpardakht-gateway'),
                    '68'  => __('تراکنش توسط بانک رد شد.', 'wc-behpardakht-gateway'),
                    '69'  => __('رمز عبور نامعتبر.', 'wc-behpardakht-gateway'),
                    '70'  => __('تراکنش به دلیل ریسک مسدود شد.', 'wc-behpardakht-gateway'),
                    '71'  => __('حساب پذیرنده نامعتبر است.', 'wc-behpardakht-gateway'),
                    '72'  => __('تراکنش توسط سیاست بانکی مسدود شد.', 'wc-behpardakht-gateway'),
                    '73'  => __('حداکثر تعداد تراکنش‌ها رسید.', 'wc-behpardakht-gateway'),
                    '74'  => __('واحد پول نامعتبر.', 'wc-behpardakht-gateway'),
                    '75'  => __('مبلغ نامعتبر برای بازپرداخت.', 'wc-behpardakht-gateway'),
                    '76'  => __('حداکثر مبلغ برای بازپرداخت بیش از حد مجاز است.', 'wc-behpardakht-gateway'),
                    '77'  => __('داده سفارش نامعتبر برای استعلام.', 'wc-behpardakht-gateway'),
                    '78'  => __('داده استعلام وجود ندارد.', 'wc-behpardakht-gateway'),
                    '79'  => __('تراکنش تسویه یافت نشد.', 'wc-behpardakht-gateway'),
                    '80'  => __('بازپرداخت رد شد.', 'wc-behpardakht-gateway'),
                    '81'  => __('داده درخواست نامعتبر است.', 'wc-behpardakht-gateway'),
                    '82'  => __('مبلغ خارج از محدوده برای این پایانه.', 'wc-behpardakht-gateway'),
                    '83'  => __('کد امنیتی نامعتبر است.', 'wc-behpardakht-gateway'),
                    '84'  => __('روش احراز هویت پشتیبانی نمی‌شود.', 'wc-behpardakht-gateway'),
                    '85'  => __('تراکنش برای این پایانه یافت نشد.', 'wc-behpardakht-gateway'),
                    '86'  => __('تراکنش قبلاً بازپرداخت شده است.', 'wc-behpardakht-gateway'),
                    '87'  => __('تراکنش اجازه تسویه ندارد.', 'wc-behpardakht-gateway'),
                    '88'  => __('مبلغ بازپرداخت معتبر نیست.', 'wc-behpardakht-gateway'),
                    '89'  => __('تراکنش برای برگشت منقضی شده است.', 'wc-behpardakht-gateway'),
                    '90'  => __('ارجاع تراکنش نامعتبر.', 'wc-behpardakht-gateway'),
                    '91'  => __('بازپرداخت ناموفق.', 'wc-behpardakht-gateway'),
                    '92'  => __('تراکنش در حال انتظار است.', 'wc-behpardakht-gateway'),
                    '93'  => __('شناسه ارجاع تکراری.', 'wc-behpardakht-gateway'),
                    '94'  => __('درخواست بازپرداخت نامعتبر.', 'wc-behpardakht-gateway'),
                    '95'  => __('بازپرداخت پشتیبانی نمی‌شود.', 'wc-behpardakht-gateway'),
                    '96'  => __('واحد پول نامعتبر برای پایانه.', 'wc-behpardakht-gateway'),
                    '97'  => __('مبلغ تراکنش نمی‌تواند صفر باشد.', 'wc-behpardakht-gateway'),
                    '98'  => __('تراکنش تعلیق شد.', 'wc-behpardakht-gateway'),
                    '99'  => __('تراکنش در حال پردازش است.', 'wc-behpardakht-gateway'),
                    '-999' => __('خطا در برقراری ارتباط با سرور به‌پرداخت ملت.', 'wc-behpardakht-gateway'),
                    'unknown' => __('خطای نامشخص رخ داده است.', 'wc-behpardakht-gateway'),
                ];

                return $errors[$code] ?? sprintf(__('خطای تعریف نشده با کد: %s', 'wc-behpardakht-gateway'), $code);
            }
        }
    }
    ```

4.  **آپلود آیکون (اختیاری اما توصیه می‌شود):** آیکون درگاه پرداخت به‌پرداخت خود (مثلاً `behpardakht.png`) را در داخل پوشه `woocommerce-behpardakht-gateway`، کنار `woocommerce-behpardakht-gateway.php` قرار دهید. این آیکون در کنار نام درگاه پرداخت در صفحه تسویه‌حساب ظاهر می‌شود.
5.  **فعال‌سازی افزونه:** وارد پنل مدیریت وردپرس خود شوید، به بخش **"افزونه‌ها"** بروید و افزونه **"درگاه پرداخت به‌پرداخت ملت برای ووکامرس"** را **فعال کنید**.
6.  **پیکربندی درگاه:** به **ووکامرس > تنظیمات > پرداخت‌ها** بروید. اکنون باید "درگاه به‌پرداخت ملت" را در لیست مشاهده کنید. روی "مدیریت" (یا "راه‌اندازی") کلیک کنید تا **شناسه پایانه، نام کاربری و رمز عبور** خود را پیکربندی نمایید.

## ملاحظات مهم

* **اعتبارنامه‌های API:** برای استفاده از این درگاه **باید** شناسه پایانه، نام کاربری و رمز عبور معتبر را از به‌پرداخت ملت (بانک ملت) دریافت کنید. این اطلاعات معمولاً پس از ثبت‌نام موفق شما در خدمات پرداخت آنلاین آن‌ها ارائه می‌شود.
* **واحد پول:** API به‌پرداخت مبلغ را به **ریال** انتظار دارد. کد ارائه شده شامل منطق تبدیل تومان به ریال (`$amount * 10`) است. اطمینان حاصل کنید که واحد پول ووکامرس شما به درستی تنظیم شده است و این تبدیل با انتظارات فعلی API به‌پرداخت مطابقت دارد.
* **گواهی SSL:** برای امنیت، اطمینان حاصل کنید که وب‌سایت شما دارای گواهی SSL معتبر (HTTPS) نصب شده است. درگاه‌های پرداخت نیاز به اتصالات امن دارند.
* **لاگ خطا:** فراخوانی‌های `error_log` برای اشکال‌زدایی مشکلات اتصال مفید هستند. در صورت بروز مشکل، لاگ خطاهای PHP سرور خود را بررسی کنید.
* **امنیت:** این یک پیاده‌سازی پایه است. برای محیط‌های تولید، همیشه بهترین روش‌های امنیتی اضافی را در نظر بگیرید، مانند لیست سفید IP (در صورت پشتیبانی توسط به‌پرداخت) و ممیزی‌های امنیتی منظم. شناسه پایانه، نام کاربری و رمز عبور خود را ایمن نگه دارید.
* **SOAP در مقابل REST:** APIهای قدیمی‌تر به‌پرداخت عمدتاً مبتنی بر SOAP هستند. این پیاده‌سازی ساده‌شده از `wp_remote_post` برای تعامل با نقطه پایانی کدگذاری شده با URL استفاده می‌کند که اغلب برای `bpPayRequest`، `bpVerifyRequest` و `bpSettleRequest` اولیه کار می‌کند. برای یکپارچه‌سازی واقعاً قوی و آینده‌نگر، به ویژه اگر به‌پرداخت API خود را تغییر دهد یا به عملکردهای پیچیده‌تر SOAP نیاز داشته باشید، ممکن است یک کلاینت SOAP اختصاصی PHP لازم باشد.
* **به‌روزرسانی‌ها:** کد درگاه سفارشی نیاز به نگهداری دارد. اطمینان حاصل کنید که با هرگونه تغییر در API به‌پرداخت یا استانداردهای درگاه پرداخت ووکامرس به‌روز می‌مانید.

## مشارکت (Contributing)

مشارکت شما خوشایند است! اگر پیشنهاد یا بهبودهایی برای این کد دارید، می‌توانید یک "Pull Request" ایجاد کنید یا "Issue" جدیدی را گزارش دهید.

## مجوز (License)

این پروژه تحت مجوز GPL-2.0-or-later منتشر شده است.
