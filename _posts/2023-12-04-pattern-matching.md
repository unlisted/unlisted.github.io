---
layout: post
title:  "Structured Pattern Matching Example"
date: 2023-12-04
categories: code pyton
---
Structured Pattern Matching was introduced in [PEP-634](https://peps.python.org/pep-0634/), tutorial available in [PEP-636](https://peps.python.org/pep-0636/). There's a lot that can be done with this, I've found it very useful in event processing, dealing with structured data.

Here's an example, idealized from a real-world use case:

Let's say you have a service that expects to process some data that looks like this
{% highlight json %}
{
    "item": {
        "item_type": "can be one of PAYMENT, CHARGE, REFUND",
        "item_status": "can be one of VALID, AUTHORIZED"
    },
    "other_data": {
        ...
    },
    "received_at": "12-12-2023T00:00:00-05",
    "created_at": "12-12-2023T00:00:00-05"
}
{% endhighlight %}

And you deserialized to a Python object (I like to use [Pydantic](https://docs.pydantic.dev/latest/) for this type of thing). Now you can implement some processing predicated on type and status
{% highlight python %}
# The model structure could look something like this
class ItemType(str, Enum):
    PAYMENT = "payment"
    CHARGE = "charge"
    REFUND = "refund"


class ItemStatus(str, Enum):
    VALID = "valid"
    AUTHORIZED = "authorized"


class Item(BaseModel):
    item_type: ItemType
    item_status: ItemStatus


class IncomingPayload(BaseModel):
    other_data: dict[str, str]
    item: Item
    received_at: datetime
    created_at: datetime


# payload is instance of IncomingPayload
match payload.item.json():
    case {
        "item_type": ItemType.PAYMENT,
        "item_status": ItemStatus.VALID
    }:
        # handle the valid payment case
        ...
    case {
        "item_type": ItemType.CHARGE,
        "item_status": ItemStatus.AUTHORIZED
    }:
        # handle the invalid charge case
        ...
    case _: ... # error, log, etc...
{% endhighlight %}
