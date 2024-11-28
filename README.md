 #the formula for discount amount (currency)in customer object
(Discount_Percent__c / 100) * Total_Amount__c

#FoodOptionTriggerHandler
public class FoodOptionTriggerHandler {

    /**
     * Method to update hotel information based on food option changes.
     *
     * @param newFoodOptions  List of new Food_Option__c records (Trigger.new context).
     * @param oldFoodOptions  List of old Food_Option__c records (Trigger.old context).
     * @param operation       String representing the trigger operation ('AFTER_INSERT', 'AFTER_UPDATE', 'AFTER_DELETE').
     */
    public static void updateHotelInformation(List<Food_Option__c> newFoodOptions, List<Food_Option__c> oldFoodOptions, String operation) {
        Set<Id> hotelIdsToUpdate = new Set<Id>();

        // Collect unique Hotel IDs from new Food Options (insert or update)
        if (newFoodOptions != null) {
            for (Food_Option__c foodOption : newFoodOptions) {
                if (foodOption.Hotel__c != null) {
                    hotelIdsToUpdate.add(foodOption.Hotel__c);
                }
            }
        }

        // Collect unique Hotel IDs from old Food Options (update or delete)
        if (oldFoodOptions != null) {
            for (Food_Option__c foodOption : oldFoodOptions) {
                if (foodOption.Hotel__c != null) {
                    hotelIdsToUpdate.add(foodOption.Hotel__c);
                }
            }
        }

        if (hotelIdsToUpdate.isEmpty()) {
            return;
        }

        // Query the affected Hotel records
        List<Hotel__c> hotelsToUpdate = [SELECT Id, TotalFoodOptions__c FROM Hotel__c WHERE Id IN :hotelIdsToUpdate];

        // Recalculate the total food options count for each hotel
        for (Hotel__c hotel : hotelsToUpdate) {
            Integer totalFoodOptions = [SELECT COUNT() FROM Food_Option__c WHERE Hotel__c = :hotel.Id];
            hotel.TotalFoodOptions__c = totalFoodOptions;
        }

        // Update the Hotel records with the new total count
        if (!hotelsToUpdate.isEmpty()) {
            update hotelsToUpdate;
        }
    }
}

#FoodOptionTrigger
trigger FoodOptionTrigger on Food_Option__c (after insert, after update, after delete) {

    // After Insert
    if (Trigger.isInsert && Trigger.isAfter) {
        FoodOptionTriggerHandler.updateHotelInformation(Trigger.new, null, 'insert');
    }

    // After Update
    if (Trigger.isUpdate && Trigger.isAfter) {
        FoodOptionTriggerHandler.updateHotelInformation(Trigger.new, Trigger.old, 'update');
    }

    // After Delete
    if (Trigger.isDelete && Trigger.isAfter) {
        FoodOptionTriggerHandler.updateHotelInformation(null, Trigger.old, 'delete');
    }
}


#FoodOptionTriggerTest
@isTest
public class FoodOptionTriggerTest {

    @testSetup
    static void setupTestData() {
        // Create a test Hotel record
        Hotel__c testHotel1 = new Hotel__c(Name = 'Hotel A');
        insert testHotel1;

        // Create Food Option records for testHotel1
        List<Food_Option__c> foodOptions = new List<Food_Option__c>{
            new Food_Option__c(Item_Name__c = 'Pizza', Hotel__c = testHotel1.Id),
            new Food_Option__c(Item_Name__c = 'Burger', Hotel__c = testHotel1.Id)
        };
        insert foodOptions;
    }

    @isTest
    static void testAfterInsert() {
        // Fetch the Hotel record before insert
        Hotel__c testHotel = [SELECT Id, TotalFoodOptions__c FROM Hotel__c WHERE Name = 'Hotel A' LIMIT 1];
        System.assertEquals(2, testHotel.TotalFoodOptions__c, 'Initial count should be 2');

        // Insert a new Food Option
        Food_Option__c newFoodOption = new Food_Option__c(Item_Name__c = 'Pasta', Hotel__c = testHotel.Id);
        insert newFoodOption;

        // Verify the count after insert
        testHotel = [SELECT Id, TotalFoodOptions__c FROM Hotel__c WHERE Id = :testHotel.Id];
        System.assertEquals(3, testHotel.TotalFoodOptions__c, 'Count should increase to 3 after insert');
    }
}


#FlightReminderScheduledJob
public class FlightReminderScheduledJob implements Schedulable {


    public void execute(SchedulableContext sc) {

        sendFlightReminders();

    }


    private void sendFlightReminders() {

        // Query for flights departing within the next 24 hours

        List<Flight__c> upcomingFlights = [SELECT Id, Name, DepartureDateTime__c FROM Flight__c

                                           WHERE DepartureDateTime__c >= :DateTime.now()

                                           AND DepartureDateTime__c <= :DateTime.now().addDays(1)];


        for (Flight__c flight : upcomingFlights) {

            // Customize the logic to send reminder emails

            // For this example, we'll print a log message; replace this with your email sending logic.

            System.debug('Sending reminder email for Flight ' + flight.Name + ' to ' + flight.ContactEmail__c);


            // Example: Send email using Messaging.SingleEmailMessage

            Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();

            email.setToAddresses(new List<String>{ flight.ContactEmail__c });

            email.setSubject('Flight Reminder: ' + flight.Name);

            email.setPlainTextBody('This is a reminder for your upcoming flight ' + flight.Name +

                                   ' departing on ' + flight.DepartureDateTime__c);

            Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ email });

        }

    }

}

#Executable window

String cronExp = '0 0 6 * * ?';


System.schedule('FlightReminderJob', cronExp, new FlightReminderScheduledJob());
