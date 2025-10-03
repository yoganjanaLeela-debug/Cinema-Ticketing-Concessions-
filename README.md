import java.util.*;

/*
 * CinemaTicketing.java
 * Single-file Java console application implementing a simple Cinema Ticketing & Concessions system.
 * - Menu-driven program
 * - Prevents double booking
 * - Payment confirmation before finalizing booking
 * - Cancellations release seats immediately
 * - Concessions can be linked to booking or placed separately
 *
 * Note: For simplicity this is an in-memory application (no persistence).
 */

public class CinemaTicketing {
    private static final Scanner scanner = new Scanner(System.in);
    private static final CinemaSystem system = new CinemaSystem();

    public static void main(String[] args) {
        seedSampleData();
        boolean running = true;
        while (running) {
            printMenu();
            String choice = scanner.nextLine().trim();
            switch (choice) {
                case "1": addMovie(); break;
                case "2": scheduleShow(); break;
                case "3": addCustomer(); break;
                case "4": bookSeats(); break;
                case "5": cancelBooking(); break;
                case "6": orderConcessions(); break;
                case "7": displayShows(); break;
                case "8": running = false; System.out.println("Exiting. Goodbye!"); break;
                default: System.out.println("Invalid option. Try again.");
            }
        }
    }

    private static void printMenu() {
        System.out.println("\n=== Cinema Ticketing System ===");
        System.out.println("1. Add Movie");
        System.out.println("2. Schedule Show");
        System.out.println("3. Add Customer");
        System.out.println("4. Book Seats");
        System.out.println("5. Cancel Booking");
        System.out.println("6. Order Concessions");
        System.out.println("7. Display Shows & Availability");
        System.out.println("8. Exit");
        System.out.print("Choose an option: ");
    }

    private static void addMovie() {
        System.out.print("Movie title: ");
        String title = scanner.nextLine().trim();
        if (title.isEmpty()) { System.out.println("Title cannot be empty."); return; }
        System.out.print("Duration minutes: ");
        String d = scanner.nextLine().trim();
        try {
            int dur = Integer.parseInt(d);
            Movie m = new Movie(title, dur);
            system.addMovie(m);
            System.out.println("Added movie: " + m.getTitle() + " (id=" + m.getId() + ")");
        } catch (NumberFormatException e) { System.out.println("Invalid number for duration."); }
    }

    private static void scheduleShow() {
        System.out.println("Available movies:");
        for (Movie m : system.getMovies()) System.out.println(m.getId() + ": " + m.getTitle());
        System.out.print("Movie ID: ");
        String mid = scanner.nextLine().trim();
        Movie movie = system.findMovieById(mid);
        if (movie == null) { System.out.println("Movie not found."); return; }

        System.out.print("Screen name: ");
        String screenName = scanner.nextLine().trim();
        System.out.print("Rows (e.g. 5): ");
        int rows = readInt(); if (rows <= 0) { System.out.println("Invalid rows."); return; }
        System.out.print("Seats per row (e.g. 8): ");
        int cols = readInt(); if (cols <= 0) { System.out.println("Invalid seats per row."); return; }
        System.out.print("Show time (e.g. 2025-10-04 18:30): ");
        String time = scanner.nextLine().trim();
        System.out.print("Base ticket price (e.g. 150): ");
        double price = readDouble();

        Screen screen = new Screen(screenName, rows, cols);
        Show show = new Show(movie, screen, time, price);
        system.addShow(show);
        System.out.println("Scheduled show id=" + show.getId() + " for movie '" + movie.getTitle() + "' on screen " + screen.getName());
    }

    private static void addCustomer() {
        System.out.print("Customer name: ");
        String name = scanner.nextLine().trim();
        if (name.isEmpty()) { System.out.println("Name cannot be empty."); return; }
        System.out.print("Phone (digits): ");
        String phone = scanner.nextLine().trim();
        if (!phone.matches("\\d{6,15}")) { System.out.println("Invalid phone."); return; }
        Customer c = new Customer(name, phone);
        system.addCustomer(c);
        System.out.println("Added customer id=" + c.getId());
    }

    private static void bookSeats() {
        System.out.println("Available shows:");
        for (Show s : system.getShows()) System.out.println(s.getId() + ": " + s.getSummary());
        System.out.print("Show ID: ");
        String sid = scanner.nextLine().trim();
        Show show = system.findShowById(sid);
        if (show == null) { System.out.println("Show not found."); return; }

        System.out.print("Customer ID (or 'new' to create): ");
        String cid = scanner.nextLine().trim();
        Customer customer = null;
        if (cid.equalsIgnoreCase("new")) {
            addCustomer();
            customer = system.getLastCustomer();
        } else {
            customer = system.findCustomerById(cid);
            if (customer == null) { System.out.println("Customer not found."); return; }
        }

        show.printAvailability();
        System.out.print("Enter seat codes separated by commas (e.g. A1,A2): ");
        String seatInput = scanner.nextLine();
        String[] codes = seatInput.split(",");
        List<Seat> chosen = new ArrayList<>();
        for (String raw : codes) {
            String code = raw.trim().toUpperCase();
            if (code.isEmpty()) continue;
            Seat seat = show.getSeatByCode(code);
            if (seat == null) { System.out.println("Seat not found: " + code); return; }
            if (!seat.isAvailable()) { System.out.println("Seat not available: " + code); return; }
            chosen.add(seat);
        }

        System.out.println("Selected seats: " + chosen);
        double subtotal = chosen.size() * show.getBasePrice();
        double tax = subtotal * 0.12; // 12% tax
        double total = subtotal + tax;
        System.out.printf("Fare: %.2f, Taxes: %.2f, Total: %.2f\n", subtotal, tax, total);

        System.out.print("Confirm payment? (yes/no): ");
        String pay = scanner.nextLine().trim().toLowerCase();
        if (!pay.equals("yes")) { System.out.println("Payment not confirmed. Booking aborted."); return; }

        // finalize booking: mark seats as booked and create booking object
        synchronized (show) {
            // double-check availability inside synchronized block
            for (Seat s : chosen) if (!s.isAvailable()) { System.out.println("Seat just got booked: " + s.getCode()); return; }
            for (Seat s : chosen) s.setAvailable(false);
            Booking booking = new Booking(customer, show, chosen);
            system.addBooking(booking);
            // Payment recorded
            Payment p = new CardPayment(total, "CONFIRMED");
            booking.setPayment(p);
            System.out.println("Booking confirmed. Receipt:");
            booking.printReceipt();
        }
    }

    private static void cancelBooking() {
        System.out.print("Booking ID to cancel: ");
        String bid = scanner.nextLine().trim();
        Booking booking = system.findBookingById(bid);
        if (booking == null) { System.out.println("Booking not found."); return; }
        // Simple cancellation policy: allow cancellation anytime
        synchronized (booking.getShow()) {
            for (Seat s : booking.getSeats()) s.setAvailable(true);
            booking.setCancelled(true);
            System.out.println("Booking cancelled. Seats released.");
        }
    }

    private static void orderConcessions() {
        System.out.print("Link to booking ID (or leave blank for standalone): ");
        String bid = scanner.nextLine().trim();
        Booking booking = null;
        if (!bid.isEmpty()) {
            booking = system.findBookingById(bid);
            if (booking == null) { System.out.println("Booking not found."); return; }
        }
        ConcessionMenu.printMenu();
        List<ConcessionItem> items = new ArrayList<>();
        while (true) {
            System.out.print("Enter item code (or 'done'): ");
            String code = scanner.nextLine().trim().toUpperCase();
            if (code.equals("DONE")) break;
            ConcessionItem it = ConcessionMenu.findByCode(code);
            if (it == null) { System.out.println("Invalid code."); continue; }
            System.out.print("Quantity: ");
            int q = readInt(); if (q <= 0) { System.out.println("Invalid quantity."); continue; }
            ConcessionItem copy = new ConcessionItem(it.getCode(), it.getName(), it.getPrice(), q);
            items.add(copy);
        }
        if (items.isEmpty()) { System.out.println("No items ordered."); return; }
        ConcessionOrder order = new ConcessionOrder(items);
        if (booking != null) booking.setConcessionOrder(order);
        system.addConcessionOrder(order);
        System.out.println("Concession receipt:");
        order.printReceipt();
    }

    private static void displayShows() {
        for (Show s : system.getShows()) {
            System.out.println(s.getId() + ": " + s.getSummary());
            s.printAvailability();
            System.out.println();
        }
    }

    private static int readInt() {
        String t = scanner.nextLine().trim();
        try { return Integer.parseInt(t); } catch (Exception e) { return -1; }
    }
    private static double readDouble() {
        String t = scanner.nextLine().trim();
        try { return Double.parseDouble(t); } catch (Exception e) { return 0.0; }
    }

    private static void seedSampleData() {
        Movie m1 = new Movie("The AI Adventure", 120);
        Movie m2 = new Movie("Space Journey", 140);
        system.addMovie(m1); system.addMovie(m2);
        Screen s1 = new Screen("Screen 1", 5, 6);
        Show sh1 = new Show(m1, s1, "2025-10-05 18:00", 200);
        Show sh2 = new Show(m2, new Screen("Screen 2", 6, 8), "2025-10-05 20:00", 250);
        system.addShow(sh1); system.addShow(sh2);
        system.addCustomer(new Customer("Alice","9876543210"));
        system.addCustomer(new Customer("Bob","9123456789"));

        // seed concession menu
        ConcessionMenu.addItem(new ConcessionItem("C1","Popcorn (Small)", 80.0, 1));
        ConcessionMenu.addItem(new ConcessionItem("C2","Popcorn (Large)", 120.0, 1));
        ConcessionMenu.addItem(new ConcessionItem("D1","Cold Drink", 70.0, 1));
        ConcessionMenu.addItem(new ConcessionItem("S1","Samosa", 50.0, 1));
    }
}

/* -------------------------- Domain & System Classes -------------------------- */

class CinemaSystem {
    private final List<Movie> movies = new ArrayList<>();
    private final List<Show> shows = new ArrayList<>();
    private final List<Customer> customers = new ArrayList<>();
    private final List<Booking> bookings = new ArrayList<>();
    private final List<ConcessionOrder> concessionOrders = new ArrayList<>();

    void addMovie(Movie m) { movies.add(m); }
    List<Movie> getMovies() { return Collections.unmodifiableList(movies); }
    Movie findMovieById(String id) { return movies.stream().filter(m -> m.getId().equals(id)).findFirst().orElse(null); }

    void addShow(Show s) { shows.add(s); }
    List<Show> getShows() { return Collections.unmodifiableList(shows); }
    Show findShowById(String id) { return shows.stream().filter(s -> s.getId().equals(id)).findFirst().orElse(null); }

    void addCustomer(Customer c) { customers.add(c); }
    Customer getLastCustomer() { return customers.isEmpty() ? null : customers.get(customers.size()-1); }
    Customer findCustomerById(String id) { return customers.stream().filter(c -> c.getId().equals(id)).findFirst().orElse(null); }

    void addBooking(Booking b) { bookings.add(b); }
    Booking findBookingById(String id) { return bookings.stream().filter(b -> b.getId().equals(id)).findFirst().orElse(null); }

    void addConcessionOrder(ConcessionOrder o) { concessionOrders.add(o); }
}

class Movie {
    private static int counter = 1;
    private final String id;
    private String title;
    private int durationMin;

    public Movie(String title, int durationMin) {
        this.id = "M" + (counter++);
        this.title = title;
        this.durationMin = durationMin;
    }
    public String getId() { return id; }
    public String getTitle() { return title; }
    public int getDurationMin() { return durationMin; }
}

class Screen {
    private String name;
    private int rows;
    private int cols;

    public Screen(String name, int rows, int cols) {
        this.name = name;
        this.rows = rows;
        this.cols = cols;
    }
    public String getName() { return name; }
    public int getRows() { return rows; }
    public int getCols() { return cols; }
}

class Show {
    private static int counter = 1;
    private final String id;
    private Movie movie;
    private Screen screen;
    private String time; // for simplicity a string
    private double basePrice;
    private Map<String, Seat> seatMap = new LinkedHashMap<>();

    public Show(Movie movie, Screen screen, String time, double basePrice) {
        this.id = "S" + (counter++);
        this.movie = movie;
        this.screen = screen;
        this.time = time;
        this.basePrice = basePrice;
        initSeats();
    }

    private void initSeats() {
        for (int r = 0; r < screen.getRows(); r++) {
            char rowChar = (char) ('A' + r);
            for (int c = 1; c <= screen.getCols(); c++) {
                String code = "" + rowChar + c;
                seatMap.put(code, new Seat(code, true));
            }
        }
    }

    public String getId() { return id; }
    public Movie getMovie() { return movie; }
    public Screen getScreen() { return screen; }
    public String getTime() { return time; }
    public double getBasePrice() { return basePrice; }

    public String getSummary() {
        return movie.getTitle() + " @ " + time + " (" + screen.getName() + ")";
    }

    public void printAvailability() {
        System.out.println("Seat availability for show " + id + ":");
        int col = screen.getCols();
        int idx = 0;
        for (Seat s : seatMap.values()) {
            System.out.print((s.isAvailable() ? "[ ]" : "[X]") + s.getCode() + " ");
            idx++;
            if (idx % col == 0) System.out.println();
        }
        if (idx % col != 0) System.out.println();
    }

    public Seat getSeatByCode(String code) { return seatMap.get(code); }
}

class Seat {
    private String code;
    private boolean available;

    public Seat(String code, boolean available) { this.code = code; this.available = available; }
    public String getCode() { return code; }
    public boolean isAvailable() { return available; }
    public void setAvailable(boolean available) { this.available = available; }
    @Override public String toString() { return code; }
}

class Customer {
    private static int counter = 1;
    private final String id;
    private String name;
    private String phone;

    public Customer(String name, String phone) {
        this.id = "C" + (counter++);
        this.name = name; this.phone = phone;
    }
    public String getId() { return id; }
    public String getName() { return name; }
    public String getPhone() { return phone; }
}

class Booking {
    private static int counter = 1;
    private final String id;
    private Customer customer;
    private Show show;
    private List<Seat> seats;
    private Payment payment;
    private boolean cancelled = false;
    private ConcessionOrder concessionOrder;

    public Booking(Customer customer, Show show, List<Seat> seats) {
        this.id = "B" + (counter++);
        this.customer = customer; this.show = show; this.seats = new ArrayList<>(seats);
    }
    public String getId() { return id; }
    public Customer getCustomer() { return customer; }
    public Show getShow() { return show; }
    public List<Seat> getSeats() { return Collections.unmodifiableList(seats); }
    public void setPayment(Payment p) { this.payment = p; }
    public void setCancelled(boolean c) { this.cancelled = c; }
    public void setConcessionOrder(ConcessionOrder o) { this.concessionOrder = o; }

    public void printReceipt() {
        System.out.println("---- Booking Receipt ----");
        System.out.println("Booking ID: " + id);
        System.out.println("Customer: " + customer.getName() + " (" + customer.getPhone() + ")");
        System.out.println("Show: " + show.getMovie().getTitle() + " @ " + show.getTime());
        System.out.println("Seats: " + seats);
        double fare = seats.size() * show.getBasePrice();
        double tax = fare * 0.12;
        double total = fare + tax;
        System.out.printf("Fare: %.2f\nTax (12%%): %.2f\nTotal: %.2f\n", fare, tax, total);
        if (concessionOrder != null) {
            System.out.println("-- Concession linked --");
            concessionOrder.printReceipt();
        }
        System.out.println("Payment: " + (payment == null ? "PENDING" : payment.getStatus() + " (" + payment.getAmount() + ")"));
        System.out.println("-------------------------");
    }
}

/* -------------------------- Concessions & Payments -------------------------- */

class ConcessionItem {
    private String code;
    private String name;
    private double price;
    private int quantity;

    public ConcessionItem(String code, String name, double price, int quantity) {
        this.code = code; this.name = name; this.price = price; this.quantity = quantity;
    }
    public String getCode() { return code; }
    public String getName() { return name; }
    public double getPrice() { return price; }
    public int getQuantity() { return quantity; }
    public double total() { return price * quantity; }
    @Override public String toString() { return name + " x" + quantity + " (" + total() + ")"; }
}

class ConcessionOrder {
    private static int counter = 1;
    private final String id;
    private List<ConcessionItem> items;

    public ConcessionOrder(List<ConcessionItem> items) { this.id = "CO" + (counter++); this.items = new ArrayList<>(items); }
    public String getId() { return id; }
    public double subtotal() { return items.stream().mapToDouble(ConcessionItem::total).sum(); }
    public double tax() { return subtotal() * 0.05; } // 5% concession tax
    public double total() { return subtotal() + tax(); }
    public void printReceipt() {
        System.out.println("---- Concession Receipt ----");
        System.out.println("Order ID: " + id);
        items.forEach(i -> System.out.println(i));
        System.out.printf("Subtotal: %.2f\nTax (5%%): %.2f\nTotal: %.2f\n", subtotal(), tax(), total());
        System.out.println("----------------------------");
    }
}

class ConcessionMenu {
    private static final List<ConcessionItem> menu = new ArrayList<>();
    static void addItem(ConcessionItem i) { menu.add(i); }
    static void printMenu() {
        System.out.println("--- Concession Menu ---");
        for (ConcessionItem i : menu) System.out.println(i.getCode()+": " + i.getName() + " - " + i.getPrice());
        System.out.println("(Enter code like C1, quantity when prompted. Type 'done' when finished.)");
    }
    static ConcessionItem findByCode(String code) {
        return menu.stream().filter(i -> i.getCode().equalsIgnoreCase(code)).findFirst().orElse(null);
    }
}

abstract class Payment {
    private double amount;
    private String status;
    public Payment(double amount, String status) { this.amount = amount; this.status = status; }
    public double getAmount() { return amount; }
    public String getStatus() { return status; }
}

class CardPayment extends Payment {
    public CardPayment(double amount, String status) { super(amount, status); }
}

class CashPayment extends Payment {
    public CashPayment(double amount, String status) { super(amount, status); }
}

/* -------------------------- End of File -------------------------- */
