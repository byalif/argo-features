# Payment Receipt Handling Workflow Feature

## 1) Introduction

The **Payment Receipt Handling Workflow** is a critical feature for an e-commerce platform designed to efficiently and securely handle the generation, storage, and dispatch of payment receipts. The feature ensures that customers receive payment confirmation emails and that all receipt data is stored in the database for compliance and auditing purposes. The system leverages **Kafka** for asynchronous communication between microservices, ensuring scalability and fault tolerance.

The workflow includes seamless integration with **Stripe** for payment validation, **Kafka** for event streaming, **Keycloak** for secure authentication, and robust database management for storing transaction data. This feature prioritizes reliability, consistency, and customer satisfaction.

---

## 2) Project Architecture

### Microservices

1. **Payment Service**  
   Validates payment transactions through Stripe and produces Kafka events for downstream processing.

2. **Receipt Generator Service**  
   Generates receipt data, including itemized purchases, transaction IDs, and timestamps.

3. **Email Notification Service**  
   Handles the dispatch of payment receipt emails to customers.

4. **Database Service**  
   Stores all payment receipt data for future reference, compliance, and auditing.

5. **Logging and Monitoring Service**  
   Tracks the entire workflow, logs errors, and integrates with tools like Splunk for monitoring.

---

## 3) Data Flow

1. **Payment Validation**  
   Payment Service validates transactions using Stripe and produces a Kafka event upon success.

2. **Receipt Generation**  
   Receipt Generator Service consumes the Kafka event, generates a detailed receipt, and publishes another Kafka event for notification and storage.

3. **Email Dispatch**  
   Email Notification Service consumes the receipt event and sends an email to the customer with the receipt attached.

4. **Database Storage**  
   Database Service consumes the receipt event and stores the receipt data in a relational database.

5. **Logging and Monitoring**  
   All microservices send logs and metrics to the Logging and Monitoring Service for centralized monitoring.

---

# Payment Controller for ArgoPrep E-Commerce Platform

This controller handles Stripe payment interactions, email notifications, and checkout session management. The implementation includes enterprise-level security with **Keycloak** for token-based authentication and **Kafka** for asynchronous processing.

```java
package com.argoprep.payment.controller;

import java.util.List;
import java.util.stream.Collectors;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import com.argoprep.payment.dto.EmailDTO;
import com.argoprep.payment.dto.ProductDetailsDTO;
import com.argoprep.payment.dto.RequestDTO;
import com.argoprep.payment.dto.ResponseDTO;
import com.argoprep.payment.service.JwtService;
import com.argoprep.payment.service.KafkaProducer;
import com.argoprep.payment.utils.CustomerUtil;
import com.argoprep.payment.utils.ProductDAO;
import com.stripe.Stripe;
import com.stripe.exception.StripeException;
import com.stripe.model.Customer;
import com.stripe.model.LineItemCollection;
import com.stripe.model.Product;
import com.stripe.model.checkout.Session;
import com.stripe.param.checkout.SessionCreateParams;
import com.stripe.param.checkout.SessionCreateParams.LineItem.PriceData;
import com.stripe.param.checkout.SessionListLineItemsParams;

import lombok.extern.slf4j.Slf4j;

@RestController
@Slf4j
@RequestMapping("/api/payments")
public class PaymentController {

    private static final String STRIPE_API_KEY = "sk_live_your_production_key";
    private static final Logger LOGGER = LoggerFactory.getLogger(PaymentController.class);

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private JwtService jwtService;

    @Autowired
    private KafkaProducer kafkaProducer;

    @PostMapping("/email/notify")
    public ResponseDTO sendEmailNotification(
            @RequestBody String sessionId,
            @RequestHeader("Authorization") String authToken) {
        EmailDTO emailDTO;
        try {
            authToken = authToken.replace("Bearer", "").trim();
            jwtService.validateToken(authToken);

            emailDTO = fetchSessionDetails(sessionId);

            // Publish email details to Kafka for async processing
            kafkaProducer.sendEmailNotification(emailDTO);

            jwtService.invalidateToken(authToken);
            return new ResponseDTO("Email notification successfully published to Kafka topic.");
        } catch (Exception e) {
            LOGGER.error("Error while sending email notification", e);
            return new ResponseDTO("Failed to send email notification: " + e.getMessage());
        }
    }

    @PostMapping("/checkout/session")
    public String createHostedCheckoutSession(@RequestBody RequestDTO requestDTO) throws StripeException {
        Stripe.apiKey = STRIPE_API_KEY;
        String clientBaseURL = "https://argoprep-ecommerce.com";

        // Retrieve or create a Stripe customer
        Customer customer = CustomerUtil.findOrCreateCustomer(requestDTO.getCustomerEmail(), requestDTO.getCustomerName());

        String jwtToken = jwtService.generateToken(requestDTO.getCustomerName(), requestDTO.getCustomerEmail());

        // Build the Stripe checkout session
        SessionCreateParams.Builder paramsBuilder = SessionCreateParams.builder()
                .setMode(SessionCreateParams.Mode.PAYMENT)
                .setCustomer(customer.getId())
                .setSuccessUrl(clientBaseURL + "/success?session_id={CHECKOUT_SESSION_ID}&token=" + jwtToken + "&email="
                        + requestDTO.getCustomerEmail())
                .setCancelUrl(clientBaseURL + "/failure");

        for (Product product : requestDTO.getItems()) {
            paramsBuilder.addLineItem(SessionCreateParams.LineItem.builder()
                    .setQuantity(1L)
                    .setPriceData(PriceData.builder()
                            .setProductData(PriceData.ProductData.builder()
                                    .setName(product.getName())
                                    .build())
                            .setCurrency(ProductDAO.getProduct(product.getId()).getDefaultPriceObject().getCurrency())
                            .setUnitAmountDecimal(
                                    ProductDAO.getProduct(product.getId()).getDefaultPriceObject().getUnitAmountDecimal())
                            .build())
                    .build());
        }

        Session session = Session.create(paramsBuilder.build());
        return session.getUrl();
    }

    private EmailDTO fetchSessionDetails(String sessionId) throws StripeException {
        Stripe.apiKey = STRIPE_API_KEY;

        Session session = Session.retrieve(sessionId);
        Customer customer = Customer.retrieve(session.getCustomer());
        String email = customer.getEmail();

        LineItemCollection lineItems = session.listLineItems(SessionListLineItemsParams.builder().build());

        List<ProductDetailsDTO> productDetails = lineItems.getData().stream()
                .map(item -> new ProductDetailsDTO(item.getDescription(), item.getQuantity(),
                        item.getAmountTotal() / 100, item.getCurrency()))
                .collect(Collectors.toList());

        EmailDTO emailDTO = new EmailDTO();
        emailDTO.setEmail(email);
        emailDTO.setProducts(productDetails);
        return emailDTO;
    }
}
```

# ArgoPrep Email Service

This **Email Service** is responsible for sending various types of emails (e.g., purchase receipts, inquiries, and welcome messages) while integrating with the database to save relevant information. It uses **Thymeleaf** for templating, and **Spring Mail** for email dispatch.

```java
package com.argoprep.email.service;

import java.util.List;
import java.util.stream.Collectors;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;

import com.argoprep.email.dto.EmailDTO;
import com.argoprep.email.dto.QuestionDTO;
import com.argoprep.email.entity.ClientResults;
import com.argoprep.email.entity.Inquiry;
import com.argoprep.email.entity.ProductDetails;
import com.argoprep.email.entity.Receipt;
import com.argoprep.email.repository.InquiryRepository;
import com.argoprep.email.repository.ProductDetailsRepo;
import com.argoprep.email.repository.ReceiptRepository;

import jakarta.mail.internet.MimeMessage;

@Service
public class EmailService {

    private static final String UTF_8_ENCODING = "UTF-8";

    private static final Logger LOGGER = LoggerFactory.getLogger(EmailService.class);

    @Autowired
    private JavaMailSender emailSender;

    @Autowired
    private ProductDetailsRepo productRepository;

    @Autowired
    private ReceiptRepository receiptRepository;

    @Autowired
    private InquiryRepository inquiryRepository;

    @Autowired
    private TemplateEngine templateEngine;

    @Value("${spring.mail.username}")
    private String username;

    /**
     * Sends purchase receipt emails asynchronously.
     *
     * @param emailDTO The email details to be sent.
     */
    public void sendPurchaseReceiptEmail(EmailDTO emailDTO) {
        try {
            LOGGER.info("Preparing receipt email for {}", emailDTO.getEmail());

            Context context = new Context();
            context.setVariable("email", emailDTO.getEmail());
            context.setVariable("products", emailDTO.getProducts());
            String emailContent = templateEngine.process("receipt", context);

            MimeMessage message = emailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, UTF_8_ENCODING);
            helper.setPriority(1);
            helper.setSubject("Your Receipt from ArgoPrep");
            helper.setFrom(username);
            helper.setTo(emailDTO.getEmail());
            helper.setText(emailContent, true);

            emailSender.send(message);
            LOGGER.info("Receipt email sent to {}", emailDTO.getEmail());

        } catch (Exception e) {
            LOGGER.error("Failed to send receipt email to {}", emailDTO.getEmail(), e);
            throw new RuntimeException("Failed to send receipt email", e);
        }
    }

    /**
     * Sends a welcome email to new users.
     *
     * @param email The email address of the new user.
     */
    public void sendWelcomeEmail(String email) {
        try {
            LOGGER.info("Preparing welcome email for {}", email);

            Context context = new Context();
            context.setVariable("email", email);
            String emailContent = templateEngine.process("welcome", context);

            MimeMessage message = emailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, UTF_8_ENCODING);
            helper.setPriority(2);
            helper.setSubject("Welcome to ArgoPrep: Your Journey Begins");
            helper.setFrom(username);
            helper.setTo(email);
            helper.setText(emailContent, true);

            emailSender.send(message);
            LOGGER.info("Welcome email sent to {}", email);

        } catch (Exception e) {
            LOGGER.error("Failed to send welcome email to {}", email, e);
            throw new RuntimeException("Failed to send welcome email", e);
        }
    }

    /**
     * Sends inquiry emails for client questions.
     *
     * @param questionDTO The inquiry details to be sent.
     */
    public void sendInquiryEmail(QuestionDTO questionDTO) {
        try {
            LOGGER.info("Preparing inquiry email for {}", questionDTO.getEmail());

            Context context = new Context();
            context.setVariable("questions", questionDTO.getQuestions());
            context.setVariable("email", questionDTO.getEmail());
            String emailContent = templateEngine.process("inquiry", context);

            MimeMessage message = emailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, UTF_8_ENCODING);
            helper.setPriority(1);
            helper.setSubject("New Client Inquiry Submission");
            helper.setFrom(username);
            helper.setTo("support@argoprep.com");
            helper.setText(emailContent, true);

            emailSender.send(message);
            LOGGER.info("Inquiry email sent from {}", questionDTO.getEmail());

        } catch (Exception e) {
            LOGGER.error("Failed to send inquiry email from {}", questionDTO.getEmail(), e);
            throw new RuntimeException("Failed to send inquiry email", e);
        }
    }

    /**
     * Saves purchase receipt data to the database.
     *
     * @param emailDTO The email details containing receipt information.
     * @return The saved Receipt entity.
     */
    public Receipt saveReceiptToDatabase(EmailDTO emailDTO) {
        try {
            List<ProductDetails> products = emailDTO.getProducts().stream()
                    .map(dto -> new ProductDetails(dto.getProductName(), dto.getQuantity(), dto.getAmount(), dto.getCurrency()))
                    .collect(Collectors.toList());

            Receipt receipt = new Receipt(products, emailDTO.getEmail());
            products.forEach(product -> product.setReceipt(receipt));

            Receipt savedReceipt = receiptRepository.save(receipt);
            LOGGER.info("Receipt saved to database for {}", emailDTO.getEmail());
            return savedReceipt;

        } catch (Exception e) {
            LOGGER.error("Failed to save receipt to database for {}", emailDTO.getEmail(), e);
            throw new RuntimeException("Failed to save receipt to database", e);
        }
    }

    /**
     * Saves inquiry data to the database.
     *
     * @param questionDTO The inquiry details to be saved.
     * @return The saved Inquiry entity.
     */
    public Inquiry saveInquiryToDatabase(QuestionDTO questionDTO) {
        try {
            List<ClientResults> results = questionDTO.getQuestions().stream()
                    .map(dto -> new ClientResults(dto.getQuestion(), dto.getAnswer()))
                    .collect(Collectors.toList());

            Inquiry inquiry = new Inquiry(results, questionDTO.getEmail());
            results.forEach(result -> result.setInquiry(inquiry));

            Inquiry savedInquiry = inquiryRepository.save(inquiry);
            LOGGER.info("Inquiry saved to database for {}", questionDTO.getEmail());
            return savedInquiry;

        } catch (Exception e) {
            LOGGER.error("Failed to save inquiry to database for {}", questionDTO.getEmail(), e);
            throw new RuntimeException("Failed to save inquiry to database", e);
        }
    }
}
```
