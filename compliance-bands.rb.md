## module CommMandates

This code creates a custom view for Google Charts Line Chart. It takes raw compliance data, determines the range of the data, creates visual thresholds data according to range + compliance mandates, then returns an object to be applied in view template.

```ruby
module CommMandates

  def comm_mandates(data, client_authority)

    # meta setup  
    view_colors = {
      'green'     => '#00ff00',
      'yellow'     => '#ffff00',
      'orange'     => '#ff6600',
      'red'       => '#ff0000',
      'trendline' => '#000000',
      'saa'       => '#999999'
    }
    colors = []

    client = client_authority
    mandate_authority = {}
    client.each do |key, value|
      if key.include? ('authority')
        authority_key = key.sub('_com_authority', '')
        mandate_authority[authority_key] = value
      end
    end

    thresholds = {
      'b2r' => 'red', 
      'r2o' => 'orange',
      'b2o' => 'orange', 
      'o2y' => 'yellow', 
      'b2y' => 'yellow',
      'r2y' => 'yellow', 
      'y2g' => 'green', 
      'b2g' => 'green',
      'g2y' => 'yellow', 
      'y2r' => 'red', 
      'y2o' => 'orange', 
      'o2r' => 'red' 
    }

    # chart mandates extraction
    chart_data = data
    # find data min, max
    min = 0;
    max = 0;
    chart_data.each_with_index do |date, index|
      date.each do |key, value|
        next if key != 'attribute_value'
        next if !value
        if index == 0
          max = value
          min = value
        else
          if value > max
            max = value
          end
          if value < min
            min = value
          end
        end
      end
    end

    if (min * 1000) > -1 and (max * 1000) < 1
      max = 0.01
      min = -0.01
    end

    # view setup
    chart_range = max - min
    view_min = min - chart_range
    view_max = max + chart_range
    view_range = view_max - view_min

    display_format = chart_data[0]['display_format'].strip

    # All mandate ranges, by date
    chart_data.each_with_index do |date, index|
      mandate_ranges = {}
      previous_key = nil
      previous_value = nil
      xrange = nil
      date.each do |key, value|
        next if key.length != 3
        next if key.index('2') != 1
        if value
          # Ugh! Ambigous mandate bookend start/stop points
          if !previous_key
            if !value or value == 0
              arbitrary_start = -5
            else
              arbitrary_start = value * 0.5
            end
            if view_min < arbitrary_start
              xrange = (view_min..value)
            else
              xrange = (arbitrary_start..value)
            end
            if key[0] == 'r'
              mandate_ranges['b2r'] = xrange
            elsif value == 0.0 and key[0] == 'g'
              mandate_ranges['b2' + key[0]] = xrange
            else
              mandate_ranges['b2' + key[2]] = xrange
            end
            previous_value = value
            previous_key = key
            next
          end
          xrange = (previous_value..value)
          mandate_ranges[previous_key] = xrange
          previous_value = value
          previous_key = key
        end
      end
      # get/set last iteration from previous
      arbitrary_end = previous_value * 1.5
      if view_max > arbitrary_end
        xrange = (previous_value..view_max)
      else
        xrange = (previous_value..arbitrary_end)
      end
      
      mandate_ranges[previous_key] = xrange

      # Is any part of mandate range in view? Calc view percentage.
      # All others get 0..0 range value to account for chart column completiion on mandate shifts
      mandates_in_view = {}
      mandate_ranges.each do |mandate, range|
        band_begin = nil
        band_end = nil
        if range.include? (view_min)
          band_begin = view_min
        end
        if range.include? (view_max)
          if !band_begin
            band_begin = range.begin
          end
          band_end = view_max
        else
          if band_begin
            band_end = range.end
          end
        end
        if view_min <= range.begin and view_max >= range.end
          band_begin = range.begin
          band_end = range.end
        end
        if band_begin and band_end
          mandates_in_view[mandate] = (band_begin..band_end)
        else # ham fist FTW!
          mandates_in_view[mandate] = (0..0)
        end
      end

      # build view data by date, add to chart_data
      view_values = []
      mandates_in_view.each do |mandate, values|
        band_value = ((values.end - values.begin) / view_range) * 100
        row_data = {
          'authority' => mandate_authority[thresholds[mandate]],
          'authority_line' => date[mandate],
          'band_value' => band_value,
          'color' => thresholds[mandate]
        }
        view_values.push(row_data)
      end
      date['mandates'] = view_values

      # chart options color array
      if index == chart_data.length - 1
        view_values.each_with_index do |v, indx|
          colors.push(view_colors[v['color']])
          break if view_values.length < 2
        end
      end
    end

    # view factor
    if !display_format.include? 'd'
      view_min = view_min * 100
      view_max = view_max * 100
    end

    # package it all up for template
    mandates_data = {}
    mandates_data['chart_data'] = chart_data
    mandates_data['chart_options'] = {}
    view_data = {
      'min' => view_min,
      'max' => view_max,
      'colors' => colors
    }
    mandates_data['chart_options'] = view_data

    return mandates_data

  end

end

```