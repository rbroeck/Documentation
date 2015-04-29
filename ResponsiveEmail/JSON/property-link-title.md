# Property `title`

The `title` property nested inside a <a href="/support/json/property-link">`link`</a>
object allows you to provide a description of the link. Usually the link title 
appears when the user hovers the link. 

    {
        "subject" : "link example",
        "from" : "info@example.com",
        "content" : {
            "blocks" : [ {
                "type" : "button",
                "label" : "Click me",
                "link" : {
                    "url" : "http://www.dustsckr2000.com/models/P2000",
                    "title", "Yes, you can click me"
                }
            } ]
        }
    }
