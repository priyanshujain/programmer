We learned that in order for DoorDash to be able to continue to safely and effectively share data across different apps, we needed to enforce modularity of the API-accessible data between applications. Thus, for each model, we created a separate resource for each relevant application:

### customers/resources.py
```py
class CustomerOrderResource(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = (
            'id',
            'subtotal',
            'tip_amount'
            ...
        )
```

### drivers/resources.py
```py
class DriverOrderResource(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = (
            'id',
            'subtotal',
            ...
        )
```

### restaurants/resources.py
```py
class RestaurantOrderResource(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = (
            'id',
            'subtotal',
            'commission',
            ...
        )
```
It seemed a bit cumbersome at first to have three separate resource classes for the same model, but it paid off big time. Prefixing each resource with the application they’re meant to be used with (reinforced by Django’s inherent application directory structure) made it easy for our engineering team to relate the resources to the correct app.

We also namespaced our API urls to reinforce modularity between applications even further. For instance, accessing an order from the customer-facing API would use `api.doordash.com/customer/order/<id>/` whereas the driver-facing API would use `api.doordash.com/driver/order/<id>/`.

With the separate API resources and the namespaced URLs, we’re now assured that all our apps are accessing the data that they’re supposed to be seeing. Plus, by embedding the data-access policies within the resource layer, privacy is automatically enforced by the API infrastructure.

We could have used fine-grain permissions logic to filter out the information that we didn’t want to expose, but it would have created unnecessary complexity for our entire back-end. We’ve found that treating each application as its own module made our lives as developers a lot easier.

### Summery
- Don’t tie your API representation directly with your data models
- Different applications have different information needs
- Use a layer of abstraction (API resources) to hide your internal data representation from the users of your API
- Use multiple API resources for the same model (or database table if you’re not using ORM) to help enforce modularity between applications
- Namespaced URLs explicate modularity between applications
- Making data-access policy choices at the API resource layer allows privacy to be automatically enforced by the API infrastructure
- Similarly, enforcing modularity eliminates the need for complex permissions logic
- Treat your internal APIs as if they were external-facing — it will force you to stick to good RESTful design principles


#### Reference

- https://doordash.engineering/2013/10/07/implementing-rest-apis-with-embedded-privacy/