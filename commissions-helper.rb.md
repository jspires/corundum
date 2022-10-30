## module CommissionsHelper
Utilized by a proprietary Ruby on Rails real-estate application. Calculates agent and company commissions on any transaction, based on transaction details, company split specifics, and employee level commission rates.

```ruby
module CommissionsHelper

  def calculate_commission(c,t)
    if params[:rdata] and params[:rdata][:ufid]
      u = params[:rdata][:ufid]
    else
      u = {}
    end
    if params[:rdata] and params[:rdata][:commission]
      ac = params[:rdata][:commission]
    else
      ac = {}
    end

    # initial
    c = get_total_tax(c,t,ac,u)
    c = get_sales_volume_credits(c,t,ac,u)
    c = get_commission_volume_credits(c,t,ac)
    c = get_commission_agent_splits(c,t,ac,u)
    # agent 1
    c = get_agent_gross_commission(c,t,ac,'agent_1_')
    c = get_company_gross_commission(c,t,ac,'1')
    c = get_agent_taxes(c,t,ac,'1')
    c = get_agent_fees(c,t,ac,'1',u)
    c = get_agent_net(c,t,ac,'1')
    c = get_company_commission(c,t,ac,'1')
    # agent 2
    if t.are_agent_2
      c = get_agent_gross_commission(c,t,ac,'agent_2_')
      c = get_company_gross_commission(c,t,ac,'2')
      c = get_agent_taxes(c,t,ac,'2')
      c = get_agent_fees(c,t,ac,'2',u)
      c = get_agent_net(c,t,ac,'2')
      c = get_company_commission(c,t,ac,'2')
    end
    c = get_company_taxes(c,t,ac)
    c = get_company_gross_income(c,t,ac)
    csplit = get_company_split(t)
    if csplit
      c[:split_id] = csplit.id
      c = get_company_fees_paid(c,t,ac,csplit,u)
      c = get_company_distributions(c,t,ac,csplit,u)
      c = get_company_net(c,t,ac,csplit)
    else
      c.errors.add( 
        :split_id, 
        message: 'No active Company Split matches the existing values for Transaction: ' + t.id.to_s + '. Calculation aborted.'
      )
    end

    return c
   
  end


  def get_company_fees_paid(c,t,ac,s,u)
      #new
      if ac[:team_lead_fee_rate]
        ctl = ac[:team_lead_fee_rate].to_i * 0.0001
      elsif s.team_lead_1 > 0
        ctl = ac[:team_lead_fee_rate] ? ac[:team_lead_fee_rate].to_i * 0.0001 : s.team_lead_1
      elsif s.team_lead_2 > 0
        ctl = ac[:team_lead_fee_rate] ? ac[:team_lead_fee_rate].to_i * 0.0001 : s.team_lead_2
      elsif s.team_lead_3 > 0
        ctl = ac[:team_lead_fee_rate] ? ac[:team_lead_fee_rate].to_i * 0.0001 : s.team_lead_3
      else
        ctl = nil
      end
      if ctl
        if u[:team_lead_fee_rate]
          c[:team_lead_fee_rate] = ac[:team_lead_fee_rate].to_i
        else
          c[:team_lead_fee_rate] = ac[:team_lead_fee_rate] ? ac[:team_lead_fee_rate] : (ctl * 10000).to_i
        end
        c[:team_lead_fee] = c[:company_gross_commission_income] * ctl
      end
      if s[:transaction_manager] && s[:transaction_manager] > 0
        c[:transaction_fee] = u[:transaction_fee] || ac[:transaction_fee] ? ac[:transaction_fee] : s.transaction_manager.to_i * 100
      end
      c[:referral_fee] = u[:referral_fee] || ac[:referral_fee] ? ac[:referral_fee] : 0
      #recalc
      return c
    end

    def get_company_gross_income(c,t,ac)
      #new
      c[:company_gross_commission_income] = c[:company_total_commission] - c[:company_bo_tax]
      #recalc
      return c
    end

    def get_company_taxes(c,t,ac)
      c[:company_bo_tax] = c[:company_total_commission] * (c[:bo_rate] * 0.0001)
      #recalc
      return c
    end

    def get_company_commission(c,t,ac,agent)
      if agent === '1'
        c[:company_total_commission] = c['company_gross_commission_' + agent]
      else
        c[:company_total_commission] += c['company_gross_commission_' + agent]
      end
      return c
    end

    def get_agent_net(c,t,ac,agent)
      c['agent_' + agent + '_net_commission'] = c['agent_' + agent + '_gross'] - c['agent_' + agent + '_bo_tax'] - c['agent_' + agent + '_kw_cap_paid'] - c['agent_' + agent + '_kw_royalty_paid'] - c['agent_' + agent + '_transaction_fee']
      c['company_net_commission_' + agent] = c['company_gross_commission_' + agent] - c['company_tax_' + agent]
      return c
    end

    def get_agent_fees(c,t,ac,agent,u)
      c['agent_' + agent + '_kw_cap_paid'] = u['agent_' + agent + '_kw_cap_paid'] || ac['agent_' + agent + '_kw_cap_paid'] ? ac['agent_' + agent + '_kw_cap_paid'] : 0
      c['agent_' + agent + '_kw_royalty_paid'] = u['agent_' + agent + '_kw_royalty_paid'] || ac['agent_' + agent + '_kw_royalty_paid'] ? ac['agent_' + agent + '_kw_royalty_paid'] : 0
      if c['agent_' + agent + '_gross'] != 0 and c['agent_' + agent + '_split'] != 0
        c['agent_' + agent + '_transaction_fee'] = u['agent_' + agent + '_transaction_fee'] || ac['agent_' + agent + '_transaction_fee'] ? ac['agent_' + agent + '_transaction_fee'] : 27500
      else
        c['agent_' + agent + '_transaction_fee'] = 0
      end
      return c
    end

    def get_agent_taxes(c,t,ac,agent)
      c['agent_' + agent + '_bo_tax'] = (c[:bo_rate] * 0.0001) * c['agent_' + agent + '_gross' ]
      c['company_tax_' + agent] = (c[:bo_rate] * 0.0001) * c['company_gross_commission_' + agent ]
      return c
    end

    def get_company_gross_commission(c,t,ac,agent)
      c['company_gross_commission_' + agent] = c['agent_' + agent + '_commission_volume' ] - c['agent_' + agent + '_gross' ]
      return c
    end

    def get_agent_gross_commission(c,t,ac,agent)
      c[agent + 'gross'] = (c[agent + 'split'] * 0.0001) * c[agent + 'commission_volume']
      return c
    end

    def get_commission_agent_splits(c,t,ac,u)
      if @team['a1']
        c[:agent_1_split] = u[:agent_1_split] || ac[:agent_1_split] ? ac[:agent_1_split].to_i : get_agent_split(@team['a1'])
        c[:company_split_1] = c[:agent_1_split] && c[:agent_1_split] > 0 ? 10000 - c[:agent_1_split] : 10000
      end
      if @team['a2']
        c[:agent_2_split] = u[:agent_2_split] || ac[:agent_2_split] ? ac[:agent_2_split].to_i : get_agent_split(@team['a2'])
        c[:company_split_2] = c[:agent_2_split] && c[:agent_2_split] > 0 ? 10000 - c[:agent_2_split] : 10000
      end
      return c
    end

    def get_commission_volume_credits(c,t,ac)
      c[:agent_1_commission_volume] = (t.gross_commission_income * 0.01) * (c[:agent_1_credit] * 0.01)
      c[:agent_2_commission_volume] = (t.gross_commission_income * 0.01) * (c[:agent_2_credit] * 0.01)
      return c
    end

    def get_sales_volume_credits(c,t,ac,u)
      c[:agent_1_credit] = u[:agent_2_credit] ? 10000 - ac[:agent_2_credit].to_i : u[:agent_1_credit] ? ac[:agent_1_credit].to_i : ac[:agent_1_credit] ? ac[:agent_1_credit].to_i : t.are_agent_2 ? 5000 : 10000
      c[:agent_2_credit] = u[:agent_1_credit] ? 10000 - ac[:agent_1_credit].to_i : u[:agent_2_credit] ? ac[:agent_2_credit].to_i : ac[:agent_2_credit] ? ac[:agent_2_credit].to_i : t.are_agent_2 ? 5000 : 0
      c[:agent_1_sales_volume] = (t.sale_price * 0.01) * (c[:agent_1_credit] * 0.01)
      c[:agent_2_sales_volume] = (t.sale_price * 0.01) * (c[:agent_2_credit] * 0.01)
      return c
    end

    def get_total_tax(c,t,ac,u)
      c[:bo_rate] = u[:bo_rate] || ac[:bo_rate] ? ac[:bo_rate] : 200
      c[:total_tax] = (t.gross_commission_income * 0.01) * (c[:bo_rate] * 0.01)
      c[:total_commission_after_taxes_and_reductions] = t.gross_commission_income - c[:total_tax]
      return c
    end

    def get_agent_split(agent)
      split = 0
      if !agent['data'].distribution_id
        contributions = 0
        if agent['commissions']
          agent['commissions'].each do |c|
            contributions += c ? (c * 0.01) : 0
          end
        end
        asplits = AgentSplit.all.order(id: :desc).first()
        if agent['data'].threshold_1
          splits_str = nil
          if agent['data'].department == 'icon_team'
            splits_str = asplits.icon
          else
            case (agent['data'].alts_level) 
            when 'bronze'
              splits_str = asplits.bronze
            when 'silver'
              splits_str = asplits.silver
            when 'gold'
              splits_str = asplits.gold
            else
              splits_str = asplits.lead
            end
          end
          splits = splits_str.split(',')
          case (contributions) 
          when 0..agent['data'].threshold_1 - 0.01
            split = splits[0]
          when agent['data'].threshold_1..agent['data'].threshold_2 - 0.01
            split = splits[1]
          else
            split = splits[2]
          end
        end
      end
      return (split.to_f * 100).to_i
    end

    def get_company_split(t)
      filtered_split = nil
      if t.department == 'production_team' or t.department == 'icon_team'
        filtered_split = CompanySplit.where(active: true, agent: t.are_agent_1).first
        if !filtered_split
          filtered_split = CompanySplit.where(active: true, department: t.department).order(id: 'ASC').first
        end
      end
      if !filtered_split
      company_splits = CompanySplit.where(active: true).order(id: 'ASC')
        company_splits.each do |split|
          next if 'production_team,icon_team'.include? t.department
          if split.department === t.department
            if split.agent === t.are_agent_1
              if  t.lead_type and split.lead_type and split.lead_type.include? t.lead_type
                if split.department === 'commercial_team' or split.department === 'dirt_team' or split.department === 'alchemy_onsite'
                  filtered_split = split
                  buyout = false
                  if split.listback_type and split.listback_type.include? 'buyout' and t.listback_type.include? 'buyout'
                    buyout = true
                  end
                  if buyout or split.listback_type and t.listback_type and split.listback_type.include? t.listback_type
                    if !split.listback_sponsor and !t.listback_sponsor
                      filtered_split = split
                    else
                      if split.listback_sponsor and t.listback_sponsor and split.listback_sponsor == t.listback_sponsor
                        filtered_split = split
                      end
                    end
                  end
                end
              end
            end
          end
        end
      end
      return filtered_split
    end

    def get_company_distributions(c,t,ac,s,u)
      
      if ac and ac.keys.length > 0
        ac.each do |k,v|
          ak = k
          if k.include? 'distribution_' and k.include? '_rate'
            ak = k.gsub('_rate','')
            s[ak] = v
          end
        end
      end
      # c_running: fees and distributions summary
      c_running = c[:referral_fee] > 0 ? c[:company_gross_commission_income] - c[:referral_fee] : c[:company_gross_commission_income]
      puts 'c_running: ' + c_running.to_s
      ds = TeamMember.where(distribution_id: [0..4]).order(distribution_id: 'ASC').pluck(:id,:distribution_id)
      ds.each_with_index do |d,idx|
        if s[('distribution_' + (idx + 1).to_s).to_sym] > 0
          c[('distribution_' + (idx + 1).to_s + '_id').to_sym] = ds[idx][0]
        end
      end
      # distribution 1
      if s[:distribution_1] > 0
        dmatch = false
        if t[:department] === 'commercial_team' and dmatch and c[:agent_1_commission_volume]
          slot = c[:agent_1_commission_volume]
        else
          slot = (c_running * s[:distribution_1]).to_i
        end
        slot = u[:distribution_1_rate] ? (c_running * s[:distribution_1]).to_i : u[:distribution_1_net] ? ac[:distribution_1_net].to_i : (c_running * s[:distribution_1]).to_i
        if t[:department] === 'commercial_team'
          c[:distribution_1_deal] = slot
        end
        if t[:department] === 'builder_services'
          if t[:listback_type] === 'uncommitted'
            c[:distribution_1_listing] = slot
          end
          if t[:listback_type].length > 0 and !t[:listback_type].include? 'uncommitted'
            c[:distribution_1_listback] = slot
          end
        end
        c[:distribution_1_net] = slot
        c[:distribution_1_rate] = ((slot / s[:distribution_1]) / (c_running / s[:distribution_1])).round(3) * 1000
      end
      # distribution 2
      if s[:distribution_2] > 0
        slot = u[:distribution_2_rate] ? (c_running * s[:distribution_2]).to_i : u[:distribution_2_net] ? ac[:distribution_2_net].to_i : (c_running * s[:distribution_2]).to_i
        if t[:department] === 'dirt_team'
          c[:distribution_2_deal] = slot
        end
        if t[:department] === 'builder_services'
          if t[:listback_type] === 'uncommitted'
            c[:distribution_2_listing] = slot
          end
          if t[:listback_type].length > 0 and !t[:listback_type].include? 'uncommitted'
            c[:distribution_2_listback] = slot
          end
        end
        c[:distribution_2_net] = slot
        c[:distribution_2_rate] = ((slot / s[:distribution_2]) / (c_running / s[:distribution_2])).round(3) * 1000
      end
      # distribution 3 
      if s[:distribution_3] > 0
        c[:distribution_3_net] = u[:distribution_3_rate] ? (c_running * s[:distribution_3]).to_i : u[:distribution_3_net] ? ac[:distribution_3_net].to_i : (c_running * s[:distribution_3]).to_i
        c[:distribution_3_rate] = ((c[:distribution_3_net] / s[:distribution_3]) / (c_running / s[:distribution_3])).round(3) * 1000
      end
      # distribution 4
      if s[:distribution_4] > 0
        c[:distribution_4_net] = u[:distribution_4_rate] ? (c_running * s[:distribution_4]).to_i : u[:distribution_4_net] ? ac[:distribution_4_net].to_i : (c_running * s[:distribution_4]).to_i
        c[:distribution_4_rate] = ((c[:distribution_4_net] / s[:distribution_4]) / (c_running / s[:distribution_4])).round(3) * 1000
      end
      # distribution 5  
      if s[:distribution_5] > 0
        c[:distribution_5_net] = u[:distribution_5_rate] ? (c_running * s[:distribution_5]).to_i : u[:distribution_5_net] ? ac[:distribution_5_net].to_i : (c_running * s[:distribution_5]).to_i
        c[:distribution_5_rate] = ((c[:distribution_5_net] / s[:distribution_5]) / (c_running / s[:distribution_5])).round(3) * 1000
      end
      get_company_net(c,t,ac,s)
    end

    def get_company_net(c,t,ac,s)
      cnet = 0
      cnet += c.team_lead_fee || 0
      cnet += c.referral_fee || 0
      cnet += c.transaction_fee || 0
      cnet += c.distribution_1_net || 0
      cnet += c.distribution_2_net || 0
      cnet += c.distribution_3_net || 0
      cnet += c.distribution_4_net || 0
      cnet += c.distribution_5_net || 0
      c[:company_net] = (c.company_gross_commission_income - cnet).round(2)
      d_rate = 1.00 - s.distribution_1 + s.distribution_2 + s.distribution_3 + s.distribution_4 + s.distribution_5
      if t.gross_commission_income == 0 or c[:company_net] == 0
        c.company_rate = 0
      else
        c.company_rate = (c[:company_net] / (t.gross_commission_income * 0.0001)).round(half: :up).to_i 
      end
      return c
    end


end


```