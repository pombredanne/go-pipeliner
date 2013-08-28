go-pipeliner
============

Tool for creating automation-related pipelines.

This is a work in progress but the general idea is already here. Do contact me if you are interested on helping.

How to write your module (plugin).
----------------------------------

There are currently 3 types of plugins:

1. Input plugins: These plugins will always be at the start of the pipeline. From the pipeline point of view, they do not accept any input and generate output. At least one input plugin is required for a valid pipeline. Its job is to generate data to be consumed by the pipeline.
2. Filter plugins: These plugins accept input and generate output. They are completelly optional and a valid pipeline does not require any filter plugin to be present. Its job is to look at any input it receives and decide if each item should continue in the pipeline or be dropped from it (so it will never reach any output plugins).
3. Output plugins: These plugins are always at the end of a pipeline. They accept input and generate no output. At least one output plugin must be present for a valid pipeline. its job is to consume somehow the items that reach the end of the pipeline.

Note that one pipeline can have multiple plugins in each category (input, filter and output). Multiple input plugins are executed concurrently and their outputs are are multiplexed in a single channel that connects to the next phase of the pipeline. Multipe filter plugins are executed serially, meaning that if an item is filtered by filter1, filter2 will never see it. Multiple outputs are executed concurrently and each one get one copy of any item (using a built-in demultiplexer) that reach this pipeline stage.

When writting a plugin, you first decide which type it is. Once you did, you need to make sure your plugin implements the correct interface as defined in https://github.com/brunoga/go-pipeliner/blob/master/pipeline/pipeline.go (InputNode, FilterNode or OutputNode). For any types, the generic module interface defined in https://github.com/brunoga/go-modules/blob/master/modules.go (to simplify things, you can use the GenericModule defined in https://github.com/brunoga/go-modules/blob/master/generic-module.go using struct embedding (see existing plugins to see how it works) and only override methods that need to be overriden for your plugin.

Here are some of the most inportant methods that need to be implemented:

    func (m *YourModule) Parameters() *base_modules.ParameterMap

This returns a ParameterMap with the list of parameters accepted by your module and with default values set. All parameters are representd as strings and must be converted/validated when needed.

    func (m *YourModule) Configure(params *base_modules.ParameterMap) error

This takes a ParameterMap that will be used for configuring the module. Pipeliner will guarantee that only expected parameters will be here (based on what is returned by the Parameters() method above) but the actual syntax of the parameters must be validated by the module. An appropriate error should be return in case some parameter is invalid or nil if everything is ok.

    func (m *YourModule) Duplicate(specificId string) (base_modules.Module, error)

This method should create a new (non-configured) instance of your module using the given specificId. specificId is used to diferentiate multiple instances of the same module and must be unique for each module instance. Pipeliner uses the "name" field in any module configuration section in the config file as a specificId.

    func (m *YourModule) GetInputChannel() chan<- interface{}

This is only required for filter and output modules. This returns the channel that should be used to send data to the module (i.e. the module input channel).

    func (m *YourModule) SetOutputChannel(inputChannel chan<- interface{}) error

Only required for filter and input modules. This sets the channel to be used by the module to send its output data.

    func (m *YourModule) Start(waitGroup *sync.WaitGroup) error

Start doing the actual work in the module. The passed in WaitGroup should be signalled when the module completes its job.

    func (m *YourModule) Stop()

Simply abort any pending tasks and signals that the module is done doing work.

The last step, after actually writting the code for your module, is to register it so Pipeliner learns about it existence. To do that, you simply need to add a init() method to the module package (usually just after the code for the module) that will register it. For example:

    func init() {
	    pipeliner_modules.RegisterPipelinerFilterModule(NewYourModule(""))
    }

NewYourModule creates an unconfigured instance of your module without a specificId (the empty specificId parameter is considered the default instance). Note that there are also RegisterPipelinerInputModule and RegisterPipelinerOutputModule to be used for each module type.

How to use your module.
-----------------------

After you create your module and assumind it is working as expected, using it is simply a matter of referencing it in the config file. For example, lets assume "YourModule" is an input module with parameters "parameter_a" and "parameter_b". In the config file you would have something like this:

    - pipeline:
        # Pipeline is named test-pipeline
        name: test-pipeline
        # Configure input modules.
        input:
          # Your module set its genericId as being "my-module".  
          - my_module:
              parameter_a: 10
              parajmeter_b: true
    [...]

I guess this is good enough as an introduction. I will try to improve this whenever I have time. Feel free to make suggestions or ask questions.

