---
author: ruanshiron
pubDatetime: 2023-11-09T14:22:00Z
title: Bảy mẫu thiết kế đối tượng trong Ruby on Rails
postSlug: bay-mau-thiet-ke-doi-tuong-trong-ruby-on-rails
tags:
  - code
  - rails
  - ruby
description: Mẫu thiết kế trong Rails hay được sử dụng
---

## Service Objects

Sử dụng khi gặp các phương thức

- phức tạp
- sử dụng API bên thứ ba
- không phụ thuộc vào một model nào
- sử dụng với một số model khác nhau

```ruby
class ChargesController < ApplicationController
 def create
   CheckoutService.new(params).call
   redirect_to charges_path
 rescue Stripe::CardError => exception
   flash[:error] = exception.message
   redirect_to new_charge_path
 end
end

class CheckoutService
 DEFAULT_CURRENCY = 'USD'.freeze

 def initialize(options = {})
   options.each_pair do |key, value|
     instance_variable_set("@#{key}", value)
   end
 end

 def call
   Stripe::Charge.create(charge_attributes)
 end

 private

 attr_reader :email, :source, :amount, :description

 def currency
   @currency || DEFAULT_CURRENCY
 end

 def amount
   @amount.to_i * 100
 end

 def customer
   @customer ||= Stripe::Customer.create(customer_attributes)
 end

 def customer_attributes
   {
     email: email,
     source: source
   }
 end

 def charge_attributes
   {
     customer: customer.id,
     amount: amount,
     description: description,
     currency: currency
   }
 end
end
```

## Value Objects

Đối tượng chỉ chứa các giá trị thường xuyên phải so sánh, chuyển đổi

Ví dụ chuyển đổi để so sánh nhiệt độ

```ruby
class AutomatedThermostaticValvesController < ApplicationController
  def heat_up
    valve.update(next_degrees: params[:degrees], next_scale: params[:scale])
    render json: { was_heat_up: valve.was_heat_up }
  end

  private

  def valve
    @valve ||= AutomatedThermostaticValve.find(params[:id])
  end
end

class AutomatedThermostaticValve < ActiveRecord::Base
  before_validation :check_next_temperature, if: :next_temperature
  after_save :launch_heater, if: :was_heat_up

  attr_accessor :next_degrees, :next_scale
  attr_reader :was_heat_up

  def temperature
    Temperature.new(degrees, scale)
  end

  def temperature=(temperature)
    assign_attributes(temperature.to_h)
  end

  def next_temperature
    Temperature.new(next_degrees, next_scale) if next_degrees.present?
  end

  private

  def check_next_temperature
    @was_heat_up = false
    if temperature < next_temperature && next_temperature <= Temperature::MAX
      self.temperature = next_temperature
      @was_heat_up = true
    end
  end

  def launch_heater
    Heater.call(temperature.kelvin_degrees)
  end
end

class Temperature
  include Comparable
  SCALES = %w(kelvin celsius fahrenheit)
  DEFAULT_SCALE = 'kelvin'

  attr_reader :degrees, :scale, :kelvin_degrees

  def initialize(degrees, scale = 'kelvin')
    @degrees = degrees.to_f
    @scale = case scale
    when *SCALES then scale
    else DEFAULT_SCALE
    end

    @kelvin_degrees = case @scale
    when 'kelvin'
      @degrees
    when 'celsius'
      @degrees + 273.15
    when 'fahrenheit'
      (@degrees - 32) * 5 / 9 + 273.15
    end
  end

  def self.from_celsius(degrees_celsius)
    new(degrees_celsius, 'celsius')
  end

  def self.from_fahrenheit(degrees_fahrenheit)
    new(degrees_celsius, 'fahrenheit')
  end

  def self.from_kelvin(degrees_kelvin)
    new(degrees_kelvin, 'kelvin')
  end

  def <=>(other)
    kelvin_degrees <=> other.kelvin_degrees
  end

  def to_h
    { degrees: degrees, scale: scale }
  end

  MAX = from_celsius(25)
end
```

## Form Objects

Sử dụng lưu trữ thông tin form phức tạp.

```ruby
class UserForm
  EMAIL_REGEX = // # Some fancy email regex

  include ActiveModel::Model
  include Virtus.model

  attribute :id, Integer
  attribute :full_name, String
  attribute :email, String
  attribute :password, String
  attribute :password_confirmation, String

  validates :full_name, presence: true
  validates :email, presence: true, format: EMAIL_REGEX
  validates :password, presence: true, confirmation: true

  attr_reader :record

  def persist
    @record = id ? User.find(id) : User.new

    if valid?
      @record.attributes = attributes.except(:password_confirmation, :id)
      @record.save!
      true
    else
      false
    end
  end
end
```

```ruby
class UsersController < ApplicationController
  def create
    @form = UserForm.new(user_params)

    if @form.persist
      render json: @form.record
    else
      render json: @form.errors, status: :unpocessably_entity
    end
  end

  private

  def user_params
    params.require(:user)
          .permit(:email, :full_name, :password, :password_confirmation)
  end
end
```

## Query Objects

Như cái tên, dùng để truy vấn

```ruby
class PopularVideoQuery
  def call(relation)
    relation
      .where(type: :video)
      .where('view_count > ?', 100)
  end
end

class ArticlesController < ApplicationController
  def index
    relation = Article.accessible_by(current_ability)
    @articles = PopularVideoQuery.new.call(relation)
  end
end
```

```ruby
class BaseQuery
  def |(other)
    ChainedQuery.new do |relation|
      other.call(call(relation))
    end
  end
end

class ChainedQuery < BaseQuery
  def initialize(&block)
    @block = block
  end

  def call(relation)
    @block.call(relation)
  end
end

class WithStatusQuery < BaseQuery
  def initialize(status)
    @status = status
  end

  def call(relation)
    relation.where(status: @status)
  end
end

query = WithStatusQuery.new(:published) | PopularVideoQuery.new
query.call(Article.all).to_sql
# "SELECT \"articles\".* FROM \"articles\" WHERE \"articles\".\"status\" = 'published' AND \"articles\".\"type\" = 'video' AND (view_count > 100)"
```

## View Objects (Serializer/Presenter)

... còn tiếp ...
