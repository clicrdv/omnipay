# Omnipay

Omnipay is a library that standardize the integration of multiple off-site payment gateways. It is heavily inspired by the excellent [omniauth](http://github.com/intridea/omniauth/).

It relies on Rack middlewares and so can be plugged in any Rack application.



## Installation

Add this line to your application's Gemfile:

    gem 'omnipay'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install omnipay



## Get Started

Let's say you want to integrate payments via mangopay for your application.

You will first need to plug an Omnipay MangoPay Gateway in your application

```ruby
# config/initializers/omnipay.rb
Rails.application.configure do |config|

  # An omnipay gateway will take two arguments :
  # 
  # * The first one is an unique identifier which will be used
  #   to generate 2 urls. One for sending the user to the payment
  #   gateway, and one for the user's return from the gateway.
  #
  # * The second one is a configuration hash. Configuration can be
  #   generic or gateway-specific, see below for more details.

  config.middleware.use(
    Omnipay::Gateway::Mangopay,
    'sandbox',
    {
      :public_key  => "azerty1234",
      :private_key => "azerty1234",
      :wallet_id   => 12345
    }
  )

end
```

This configuration will make your app respond to two urls :

 * `GET /pay/mangopay/sandbox?amount=xxxx` will forward your user to mangopay for paying the xxxx amount (in cents)
 * `GET /pay/mangopay/sandbox/callback` will be called when the payment process is complete. You will need to create a route to handle this request in your application


```ruby
# config/routes.rb
get '/pay/:gateway_name/:gateway_id/callback', :to => "payments#callback"

# app/controllers/payments_controller.rb
def callback

  # In this action you have access to the hash request.env['omnipay.response']
  # This reponse hash is independant of the chosen gateway and will look like this : 
  {
    :gateway => <Omnipay::Gateway::Mangopay> # The gateway which processed the payment.
    :amount => 1295 # The amount in cents payed by the user.
    :success => true # Was the payment successful or not.
    :error_code => :invalid_pin # An error code if the payment was not successful.
    :reference => "O-12XFD-987" # The payment's reference in the gateway platform.
    :raw => <Hash> # The raw response params from the gateway
  }

end

```


## Give context to the callback

You may want to have more informations in the callback. For example, if you have an e-commerce application and the user has multiple pending orders, you may want to know what order was just payed. You can get this by passing a `context` hash to the payment URL, which will then be accessible in the `omniauth.response` hash.

```ruby

# app/views/orders/payment.html.erb
<%= link_to '/pay/mangopay/sandbox', 
            :amount => @order.amount, 
            :context => {:order_id => @order.id} %>


# app/controllers/payments_controller.rb
def callback

  omnipay_response = request.env['omnipay.response']

  order = Order.find(omnipay_response[:context][:order_id])
  order.set_paid! if omnipay_response[:success] && omnipay_response[:amount] == order.amount

end
```


## Handle dynamic gateway configuration

The initializer is a static file only loaded at the applications's start. You may however run a SAAS where multiple users each can define its gateway configuration. A way to handle this is to use a block in the gateway configuration :

```ruby
# config/initializer/omnipay.rb

# Using this configuration, each call to /pay/mangopay/:shop_id will look 
# for a shop having this id, and will forward to its payment page. 
# The callback will still be on `/pay/mangopay/:shop_id/callback`

config.middleware.use(
  Omnipay::Gateway::Mangopay do |uid|

    shop = Shop.find(uid)

    # If no config is returned, the request is forwarded to the app, which may 404
    return unless shop && shop.has_mangopay_config?

    return {
      :public_key  => shop.mangopay_public_key,
      :private_key => shop.mangopay_private_key,
      :wallet_id   => shop.mangopay_wallet_id
    }
  end
)
```

## Global omnipay configuration

TODO ...


## Gateway configuration

TODO ...


## Payment URL options

Required arguments :
 * `amount` : The amount in cents to pay.

Optional arguments :
 * `reference` : The order reference to be used in the gateway.


## Callback options

TODO ...



## Create a new Gateway

TODO ...

```ruby
class Omnipay::Gateway::Aphone < Omnipay::Gateway::Base

  # Options which can be overriden in the config
  # and found in the :options accessor
  DEFAULT_OPTIONS = {
    :payment_url => 'https://secure.oneclicpay.com'
    :payment_method => 'POST'
  }

  # Request phase url
  def request_url
    options.payment_url
  end

  # Request phase HTTP method
  def request_method
    options.payment_method
  end

  # Request phase params
  def request_params(amount)
    MyHelper.compute_params_for(
      :public_key => options.public_key, # No default value, will have to be defined in the config
      :private_key => options.private_key,
      :amount => amount
    )
  end


  # Callback phase response hash
  def callback_hash

    if MyHelper.valid_reponse(gateway_callback_params)
      {
        :success => true,
        :amount => gateway_callback_params['amount'],
        :reference => gateway_callback_params['transactionRef']
      }
    else
      {
        :success => false,
        :error => (case gateway_callback_params['responseCode']
          when 207
          :payment_refused
          when 221
          :wrong_cvv
        )
      }
    end

  end

end
```


## Error Codes

TODO ...


## Deployment

Install the `gem-release` gem [documentation](http://github.com/svenfuchs/gem-release)

`gem bump`
`gem tag`
`gem release`


## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
