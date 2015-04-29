# Property `margin`

You may change the amount of whitespace that is reserved around a block.
The `margin` property can be used for that. The margin can either be
a numeric value, if you want to use the same margin for all four sides
around the block, or an object if you want to set the top, left, right
and bottom margins seperately.

    {
        "from" : "info@example.com",
        "subject" : "This email shows how to use margins",
        "content" : {
            "blocks" : [ {
                "type" : "image",
                "src" : "http://www.example.com/image1.png",
                "margin" : 10
            }, {
                "type" : "image",
                "src" : "http://www.example.com/image2.png",
                "margin" : {
                    "left" : 10,
                    "right" : 20,
                    "top" : 0,
                    "bottom" : 5
                }
            } ]
        }
    }

Above example shows how you can use the `margin` property to set the margins
for all four sides around a block at once, and how to use the property
to set different margins for all four sides.


## Where to use?

The `margin` property can be used for every type of block, whether it is
a <a href="/support/json/block-text">text block</a>, an 
<a href="/support/json/block-image">image block</a> or any other block: 
they all have margins.
