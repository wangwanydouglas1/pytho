"""
PYTHON PRACTICAL PROGRAMMING ASSIGNMENT
Project: Digital POS & End-of-Day Reconciliation Ledger
"""

BASE_PRICES = {
    "tomatoes": 3000,
    "onions": 2500,
    "matooke": 25000
}

WHOLESALE_PRICES = {
    "tomatoes": 2400,
    "onions": 2000,
    "matooke": 20000
}


def prompt_quantity(item_name):
    """Prompt repeatedly until a numeric quantity greater than 0 is entered."""
    unit = "bunches" if item_name == "matooke" else "kg"

    while True:
        try:
            value = float(input(f"Enter quantity ({unit}): ").strip())

            if value <= 0:
                print(f"Error: Quantity must exceed 0.0 {unit}")
                continue

            return value

        except ValueError:
            print(f"Error: Please enter a valid numeric quantity in {unit}.")


def calculate_item_price(item_type, quantity, customer_type="retail"):
    """
    Calculate the price of one produce line.

    Retail:
      - Tomatoes/onions: normal price up to 5 kg, then 5% discount
        applies only to kilograms above 5.
      - Matooke: standard price per bunch.

    Wholesale:
      - Tomatoes: 2,400 UGX/kg
      - Onions: 2,000 UGX/kg
      - Matooke: 20,000 UGX/bunch only when quantity >= 3;
        otherwise use retail price.
    """
    item_type = item_type.lower().strip()
    customer_type = customer_type.lower().strip()

    if item_type not in BASE_PRICES:
        raise ValueError("Invalid item type.")

    if quantity <= 0:
        raise ValueError("Quantity must be greater than zero.")

    # Wholesale pricing
    if customer_type == "wholesale":
        if item_type == "matooke" and quantity < 3:
            return BASE_PRICES[item_type] * quantity

        unit_price = WHOLESALE_PRICES[item_type]
        return unit_price * quantity

    # Retail pricing
    if item_type in ("tomatoes", "onions") and quantity > 5:
        first_five = 5 * BASE_PRICES[item_type]
        extra_quantity = quantity - 5
        discounted_extra = extra_quantity * BASE_PRICES[item_type] * 0.95
        return first_five + discounted_extra

    return BASE_PRICES[item_type] * quantity


def get_unit_price(item_type, quantity, customer_type):
    """Return the effective unit price for receipt display."""
    item_type = item_type.lower()
    customer_type = customer_type.lower()

    if customer_type == "wholesale":
        if item_type == "matooke" and quantity < 3:
            return BASE_PRICES[item_type]
        return WHOLESALE_PRICES[item_type]

    if item_type in ("tomatoes", "onions") and quantity > 5:
        # Effective average unit price when a retail quantity crosses 5 kg.
        return calculate_item_price(item_type, quantity, customer_type) / quantity

    return BASE_PRICES[item_type]


def apply_market_levy_and_packaging(subtotal, needs_eco_crate=False):
    """Apply 1% market levy and optional 5,000 UGX refundable crate deposit."""
    levy = subtotal * 0.01
    crate_deposit = 5000 if needs_eco_crate else 0
    return subtotal + levy + crate_deposit


def get_customer_type():
    """Prompt for a valid customer category."""
    while True:
        customer_type = input(
            "Customer Type (retail/wholesale): "
        ).strip().lower()

        if customer_type in ("retail", "wholesale"):
            return customer_type

        print("Error: Please enter retail or wholesale.")


def get_yes_no(prompt):
    """Prompt for yes/no input."""
    while True:
        answer = input(prompt).strip().lower()

        if answer in ("yes", "no"):
            return answer == "yes"

        print("Error: Please enter yes or no.")


def print_receipt(customer_name, customer_type, basket, needs_eco_crate, receipt_no):
    """Print an itemized customer receipt and return the final amount."""
    subtotal = sum(item["total"] for item in basket)
    levy = subtotal * 0.01
    crate_deposit = 5000 if needs_eco_crate else 0
    total = apply_market_levy_and_packaging(subtotal, needs_eco_crate)

    print("-" * 58)
    print(f" RECEIPT #{receipt_no}")
    print(f" Customer: {customer_name} ({customer_type.upper()})")
    print("-" * 58)

    for item in basket:
        unit_label = "bn" if item["name"] == "matooke" else "kg"
        print(
            f"{item['quantity']:.1f} {unit_label} x "
            f"{item['name'].title()} @ {item['unit_price']:,.2f} UGX "
            f"= {item['total']:,.2f} UGX"
        )

    print("-" * 58)
    print(f"Subtotal: {subtotal:,.2f} UGX")

    if needs_eco_crate:
        print(f"Reusable Crate Deposit: {crate_deposit:,.2f} UGX")

    print(f"Market Council Levy (1%): {levy:,.2f} UGX")
    print("-" * 58)
    print(f"TOTAL DUE: {total:,.2f} UGX")
    print("=" * 58)

    return total


def process_customer(receipt_number):
    """Process one customer and return the customer's final bill."""
    print("\n" + "=" * 58)
    print(" MAMA MBOGA DIGITAL STALL POS")
    print("=" * 58)

    customer_name = ""
    while not customer_name:
        customer_name = input("Customer Name: ").strip()
        if not customer_name:
            print("Error: Customer name cannot be empty.")

    customer_type = get_customer_type()
    basket = []

    while True:
        item_type = input(
            "Add Item (tomatoes / onions / matooke / done): "
        ).strip().lower()

        if item_type == "done":
            break

        if item_type not in BASE_PRICES:
            print("Error: Choose tomatoes, onions, matooke, or done.")
            continue

        quantity = prompt_quantity(item_type)
        total = calculate_item_price(item_type, quantity, customer_type)
        unit_price = get_unit_price(item_type, quantity, customer_type)

        basket.append({
            "name": item_type,
            "quantity": quantity,
            "unit_price": unit_price,
            "total": total
        })

        unit = "bunches" if item_type == "matooke" else "kg"
        print(f"-> Added {quantity:.1f} {unit} of {item_type}.")

    if not basket:
        print("Error: No items were added. Customer transaction cancelled.")
        return None

    needs_eco_crate = get_yes_no(
        "Add reusable delivery crate? (yes/no): "
    )

    return print_receipt(
        customer_name,
        customer_type,
        basket,
        needs_eco_crate,
        receipt_number
    ), customer_name


def main():
    """Run the full market session and produce the daily close report."""
    total_customers = 0
    total_revenue = 0.0
    highest_customer = ""
    highest_bill = 0.0
    receipt_number = 42

    while True:
        result = process_customer(receipt_number)

        if result is not None:
            bill, customer_name = result
            total_customers += 1
            total_revenue += bill

            if bill > highest_bill:
                highest_bill = bill
                highest_customer = customer_name

            receipt_number += 1

        serve_next = get_yes_no("Serve next customer? (yes/no): ")

        if not serve_next:
            break

    print("\n" + "=" * 16 + " DAILY MARKET CLOSE " + "=" * 16)
    print(f"Total Customers Served: {total_customers}")
    print(f"Total Revenue: {total_revenue:,.2f} UGX")

    if total_customers > 0:
        print(
            f"Star Customer: {highest_customer} "
            f"({highest_bill:,.2f} UGX)"
        )
    else:
        print("Star Customer: None")

    print("=" * 58)


if __name__ == "__main__":
    main()

