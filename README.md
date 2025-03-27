package base;

import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.testng.annotations.AfterClass;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.Test;

import java.io.*;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;


public class sendMsgToMyConnections {


    public static final String sauceEmail = System.getenv("LINKEDIN_USER_EMAIL");
    public static final String saucePassword = System.getenv("LINKEDIN_USER_PASSWORD");
    private static final String CSV_FILE = "sent_messages.csv";

    static WebDriver driver;

    public static WebDriver getDriver(){
        return driver;
    }

    @BeforeClass
    public void setUp(){
        WebDriverManager.chromedriver().setup();
        driver = new ChromeDriver();
        driver.get("https://www.linkedin.com/");
        driver.manage().timeouts().implicitlyWait(10, TimeUnit.SECONDS);
        driver.manage().window().maximize();

    }

    @AfterClass
    public void tearDown(){driver.quit();
    }


    @Test
    public void sendMessages() throws InterruptedException, IOException {

        Set<String> messagedConnections = loadMessagedConnections();


        if (isCaptchaPresent(driver)) {
            System.out.println("CAPTCHA detected! Please solve it manually.");
            Thread.sleep(30000);
        }

        try {
            // Open LinkedIn and login
            driver.get("https://www.linkedin.com/");
            driver.findElement(By.xpath("//nav/div/a[2]")).click();
            driver.findElement(By.id("username")).sendKeys(sauceEmail);
            driver.findElement(By.id("password")).sendKeys(saucePassword, Keys.RETURN);
            Thread.sleep(1000);
            randomDelay(1500, 7000);

            driver.get("https://www.linkedin.com/mynetwork/invite-connect/connections/");
            randomDelay(1000, 3000);


            FileWriter writer = new FileWriter(CSV_FILE, true);
            BufferedWriter bufferedWriter = new BufferedWriter(writer);

            List<WebElement> connections = loadAllConnections(driver);


            for (WebElement connection : connections) {
                try {
                    String name = connection.findElement(By.cssSelector("span.mn-connection-card__name.t-16.t-black.t-bold")).getText();

                    if (messagedConnections.contains(name)) {
                        System.out.println("Skipping (already messaged): " + name);
                        continue;
                    }

                    System.out.println("Messaging: " + name);
                    randomDelay(1000, 3000);

                    WebElement messageButton = connection.findElement(By.xpath(".//button[contains(@aria-label, 'message')]"));
                    messageButton.click();
                    Thread.sleep(2000);
                    randomDelay(2500, 6500);

                    WebElement messageInput = driver.findElement(By.cssSelector("div.msg-form__contenteditable"));
                    messageInput.sendKeys("Hey " + name +
                            ", I’m Dana Shafran, a QA Automation Engineer with 8+ years of experience, based in Dallas, Texas. I’m currently seeking a QA Automation Engineer role either in Dallas or remotely. If you’re aware of any relevant opportunities, I’d love to connect and discuss further. Looking forward to hearing from you! Best regards, Dana Shafran");
                    Thread.sleep(1000);
                    randomDelay(1500, 4000);

                    messageInput.sendKeys(Keys.RETURN);
                    Thread.sleep(2000);

                    try {
                        WebElement dismissButton = driver.findElement(By.cssSelector("button.artdeco-toast-item__dismiss[aria-label*='Dismiss']"));
                        dismissButton.click();
                        randomDelay(1000, 3000);
                    } catch (Exception e) {
                        System.out.println("No dismiss button found: " + e.getMessage());
                    }

                    try {
                        WebElement closeChatButton = driver.findElement(By.xpath("//button[contains(@class, 'msg-overlay-bubble-header__control artdeco-button')]"));
                        closeChatButton.click();
                    } catch (Exception e) {
                        System.out.println("No close button found: " + e.getMessage());
                    }

                    try {
                        System.out.println("Saving to CSV: " + name);
                        bufferedWriter.write(name);
                        bufferedWriter.newLine();
                        bufferedWriter.flush();
                        System.out.println("Saved successfully!");
                    } catch (IOException e) {
                        System.out.println("Error writing to CSV: " + e.getMessage());
                    }

                } catch (Exception e) {
                    System.out.println("Skipping connection: " + e.getMessage());
                }
            }

            bufferedWriter.close();


        } finally {
            driver.quit();
        }
    }

    private List<WebElement> loadAllConnections(WebDriver driver) throws InterruptedException {
        List<WebElement> connections;
        int previousCount = 0;
        int maxScrollAttempts = 15; // Limit scroll attempts to prevent infinite loop
        int attempts = 0;

        while (true) {
            connections = driver.findElements(By.cssSelector("li.mn-connection-card"));
            if (connections.size() > previousCount) {
                previousCount = connections.size();
                System.out.println("Loaded connections: " + previousCount);
                ((JavascriptExecutor) driver).executeScript("window.scrollBy(0,1000);");
                Thread.sleep(3000);
            } else {
                attempts++;
                if (attempts >= maxScrollAttempts) break;
            }
        }
        return connections;
    }


    private void scrollDown(WebDriver driver) {
        JavascriptExecutor js = (JavascriptExecutor) driver;
        js.executeScript("window.scrollBy(0,1000)");
    }


    private void randomDelay(int min, int max) throws InterruptedException {
        int delay = ThreadLocalRandom.current().nextInt(min, max);
        Thread.sleep(delay);
    }

    private boolean isCaptchaPresent(WebDriver driver) {
        try {
            driver.findElement(By.xpath("//div[contains(text(), 'Verify')]"));
            return true;
        } catch (NoSuchElementException e) {
            return false;
        }
    }

    private static Set<String> loadMessagedConnections() throws IOException {
        Set<String> messagedConnections = new HashSet<>();
        File file = new File(CSV_FILE);
        if (file.exists()) {
            BufferedReader reader = null;
            try {
                reader = new BufferedReader(new FileReader(file));
                String line;
                while ((line = reader.readLine()) != null) {
                    messagedConnections.add(line.trim());
                }
            } catch (IOException e) {
                System.out.println("Error reading CSV: " + e.getMessage());
            } finally {
                if (reader != null) {
                    try {
                        reader.close();
                    } catch (IOException e) {
                        System.out.println("Error closing CSV reader: " + e.getMessage());
                    }
                }
            }

        }
        return messagedConnections;
    }

}
