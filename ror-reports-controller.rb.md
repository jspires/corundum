## ReportsController
Fairly typical Rails controller + interactive document publish to AWS and interactive sticky/session query filters 

```ruby
class ReportsController < ApplicationController
  helper ReportsHelper
  before_action :authenticate_user!, :set_report, only: [:show, :edit, :update, :destroy]
  after_action :verify_authorized
  before_filter :permit_user, :init_session_filters
  include ReportsHelper

  # GET /reports
  # GET /reports.json
  def index
    @user_meta = get_user_meta(current_user)
    @report_types = get_report_types
    if params['did'] or params['aid'] or params['rtid'] or params['sid']
      set_session_filters
    end
    set_filter_params_from_session
    @reports = Report.select(:id, :name, :account_id, :report_status, :report_type, :report_date, :description, :signature).where(account_id: @user_meta['aids'], report_status: @user_meta['status']).order(report_date: 'DESC', account_id: 'ASC', name: 'DESC')
    authorize @reports
    if params['did'] or params['aid'] or params['rtid'] or params['sid']
      @reports = filter_by_input(@reports, params, @user_meta)
      if @reports.length == 0
        zero_filtered(params,@report_types)
      end
    end
    @reports = @reports.page(params[:page]).per(16)
  end

  # GET /reports/1
  # GET /reports/1.json
  def show
    @reports = []
    if current_user.client? and @report.report_status != 'approved'
      raise Pundit::NotAuthorizedError, "Report not authorized"
    end
    if !current_user.admin? and @report.report_status == 'deleted'
      raise Pundit::NotAuthorizedError, "Report not authorized"
    end
    authorize @report
  end

  # GET /reports/new
  def new
    @report = Report.new
    authorize @report
  end

  # GET /reports/1/edit
  def edit
    authorize @report
  end

  # POST /reports
  # POST /reports.json
  def create
    if params['report']['report_document']
      params['report'] = publish_doc_to_aws(params['report'])
    end
    @report = Report.new(report_params)
    authorize @report

    respond_to do |format|
      if @report.save
        format.html { redirect_to @report, notice: 'Report was successfully created.' }
        format.json { render :show, status: :created, location: @report }
      else
        format.html { render :new }
        format.json { render json: @report.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /reports/1
  # PATCH/PUT /reports/1.json
  def update
    is_deleted = params['report'].length < 2
    if params['report']
      if params['report']['report_document']
        params['report'] = publish_doc_to_aws(params['report'])
      end
    end
    if current_user.analyst? and !is_deleted
      params['report']['report_status'] = 0
    end
    authorize @report
    respond_to do |format|
      if @report.update(report_params)
        if is_deleted
          format.html { redirect_to reports_path, notice: 'Report was successfully deleted.' }
          format.json { render :index, status: :ok }
        else
          format.html { redirect_to @report, notice: 'Report was successfully updated.' }
          format.json { render :show, status: :ok, location: @report }
        end
      else
        if is_deleted
          format.html { render :index }
          format.json { render json: @report.errors, status: :unprocessable_entity }
        else
          format.html { render :edit }
          format.json { render json: @report.errors, status: :unprocessable_entity }
        end
      end
    end
  end

  # DELETE /reports/1
  # DELETE /reports/1.json
  def destroy
    authorize @report
    @report.destroy
    respond_to do |format|
      format.html { redirect_to reports_url, notice: 'Report was successfully destroyed.' }
      format.json { head :no_content }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_report
      @report = Report.find(params[:id])
    end

    # Never trust parameters from the scary internet, only allow the white list through.
    def report_params
      params.require(:report).permit(:name, :report_type, :account_id, :description, :html, :json, :signature, :report_date, :aws_object_key, :aws_object_bucket, :report_status)
    end

    def publish_doc_to_aws(report_params)
      uploaded_io = report_params['report_document']
      object_key_param = upload_to_aws( uploaded_io )
      if object_key_param.length > 0
        report_params['aws_object_key'] = object_key_param
        report_params['aws_object_bucket'] = 'verustank'
      end
      report_params.delete('report_document')
      return report_params
    end

    def upload_to_aws(uploaded_io)
      aws_file = uploaded_io.read
      aws_object_key = uploaded_io.original_filename
      etag = false
      if aws_file.length > 0 and aws_object_key.length > 0
        s3 = Aws::S3::Client.new(
          access_key_id: Rails.application.secrets.aws_access_key,
          secret_access_key: Rails.application.secrets.aws_secret,
          region: Rails.application.secrets.aws_region
        )
        response = s3.put_object(bucket: 'verustank', key: aws_object_key, body: aws_file)
        etag = response.etag
      end
      if etag and etag.length > 0
        return aws_object_key
      else
        return ''
      end
    end

    def init_session_filters
      session['filter_params'] = {} if !session['filter_params']
      session['filter_params']['reports_params'] = {} if !session['filter_params']['reports_params']
    end

    def set_session_filters
      reports_filters = ['did','aid','rtid','sid']
      sparams = {}
      reports_filters.each do |fparam|
        if params[fparam]
          sparams[fparam] = params[fparam]
        end
      end
      session['filter_params']['reports_params'] = sparams
    end

    def set_filter_params_from_session
      if params['clear'] and params['clear'] == 'true'
        params.delete('clear')
        set_session_filters
      end
      if session['filter_params']['reports_params'].length > 0
        session['filter_params']['reports_params'].each do |fid,fval|
          if fval
            params[fid] = fval
          end
        end
      end
    end

    def remove_all_filter_params
      params.delete('did') if params['did']
      params.delete('aid') if params['aid']
      params.delete('rtid') if params['rtid']
      params.delete('sid') if params['sid']
      set_session_filters
    end

    def get_user_meta(user) 
      user_meta = {}
      # User role
      user_meta['role'] = user.role
      # User accounts
      aids = []
      if user.accounts.length > 0
        user.accounts.each do |account|
          aids.push(account[:id])
        end
      end
      user_meta['aids'] = aids
      # User role
      status = []
      if user.client?
        status = [1]
      elsif user.admin?
        status = [nil,0,1,2]
      else
        status = [nil,0,1]
      end
      user_meta['status'] = status
      return user_meta
    end

    def get_report_types
      report_types = {}
      Report.report_types.each_with_index do |type,index|
        display_str = type[0].gsub(/_/, ' ').titleize
        display_str = display_str.gsub(/ And /, ' and ')
        report_types[type[1]] = display_str
      end
      report_types = report_types.sort_by {|k,v| v}
      return report_types
    end

    def filter_by_input(reports, params, usermeta)
      query = {}
      params.each do |param|
        param.each do |pkey, pvalue|
          next if pkey == 'controller'
          next if pkey == 'action'
          case pkey
          when 'did'
            begin
              date = Date.parse(params['did'])
            rescue
              redirect_to reports_path, :notice => "Please enter a valid date."
            ensure
              query[:report_date] = date
            end
          when 'aid'
            account_authorized = false
            usermeta['aids'].each do |account|
              if account == params['aid'].to_i
                account_authorized = true
                query[:account_id] = account
              end
            end
            if !account_authorized
              raise Pundit::NotAuthorizedError, "Account not authorized"
            end
          when 'rtid'
            query[:report_type] = params['rtid'].to_i
          when 'sid'
            # Client NOT authorized to view by status
            if usermeta['role'] == 'client'
              raise Pundit::NotAuthorizedError, "Status not authorized"
            # Admin has the all-seeing-eye
            elsif usermeta['role'] == 'admin'
              query[:report_status] = params['sid'].to_i
            # All other roles CAN NOT see 'deleted'
            else
              if params['sid'] == '2'
                raise Pundit::NotAuthorizedError, "Status not authorized"
              else
                query[:report_status] = params['sid'].to_i
              end
            end
          else
          end
        end
      end
      filtered_reports = reports.where(query)
      return filtered_reports
    end

    def zero_filtered(params,report_types)
      fparams = {}
      params.each do |pkey, pvalue|
        next if pkey == 'controller'
        next if pkey == 'action'
        fparams[pkey] = pvalue
      end
      if fparams.length > 0
        notice = 'Sorry, we cannot find any reports '
        separator = ' and '
          fparams.each_with_index do |fparam, index|
          if fparam[0] == 'did'
            search_date = Date.parse(fparam[1])
            if index == 0
              notice = notice + "with date: " + search_date.strftime("%m-%d-%Y")
            else
              notice = notice + separator + "with date: " + search_date.strftime("%m-%d-%Y")
            end
          end
          if fparam[0] == 'aid'
            account = Account.find(fparam[1].to_i)
            search_account = account.name
            if index == 0
              notice = notice + "with account: " + search_account
            else
              notice = notice + separator + "with account: " + search_account
            end
          end
          if fparam[0] == 'rtid'
            search_report_type = ''
            report_types.each do |type|
              if type[0] == fparam[1].to_i
                search_report_type = type[1]
              end
            end
            if index == 0
              notice = notice + "with report type: " + search_report_type
            else
              notice = notice + separator + "with report type: " + search_report_type
            end
          end
          if fparam[0] == 'sid'
            search_report_status = ''
            Report.report_statuses.each do |status|
              if status[1] == fparam[1].to_i
                search_report_status = status[0]
              end
            end
            if index == 0
              notice = notice + "with report status: " + search_report_status
            else
              notice = notice + separator + "with report status: " + search_report_status
            end
          end
        end
        remove_all_filter_params
        redirect_to reports_path, :notice => notice
      end
    end
end


```