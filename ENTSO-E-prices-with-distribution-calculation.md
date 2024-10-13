In ENTSO-E custom integratio, use this as price modifyer template:  
```Python
{% set s = {
    "exchange_rate": states('sensor.eur_to_pln_exchange_rate')|float,
    "VAT": 1.23,
    "peak_distribution_rate": states('input_number.dystrybucja_g12w_szczyt')|float,
    "offpeak_distribution_rate": states('input_number.dystrybucja_g12w_pozaszczyt')|float
} %}

{# Define peak periods for G12W tariff as a list of tuples (start_hour, stop_hour) #}
{% set peaks = [(6, 13), (15, 22)] %}

{# Determine if it's a weekend (offpeak all day) #}
{% set is_weekend = now().weekday() >= 5 %}

{# Check if the current time falls within any peak period #}
{% set is_peak = False %}
{% if not is_weekend %}
    {% for peak in peaks %}
        {% if now().hour >= peak[0] and now().hour < peak[1] %}
            {% set is_peak = True %}
        {% endif %}
    {% endfor %}
{% endif %}

{# Set distribution rate based on peak or offpeak time #}
{% set distribution_rate = s.peak_distribution_rate if is_peak else s.offpeak_distribution_rate %}

{# Final price calculation: convert current_price to PLN, add distribution rate, and apply VAT #}
{{ ((current_price * s.exchange_rate) + distribution_rate) * s.VAT | float }}
```
