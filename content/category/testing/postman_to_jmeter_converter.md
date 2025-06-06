Title: Converting Postman collections with env support to JMeter
Category: Testing
Date: 2025-06-06
Tags: postman, jmeter, python
Slug: postman-to-jmeter-converter
Summary: Converter written in Python 3 for Postman API collections & env into JMeter (jmx) test plans.

If you've ever had to migrate Postman API tests into JMeter for load testing, you know the drill: either you manually recreate everything (pain), or you find some hacky tool that half-works. So I wrote a script in Python to make this a bit less annoying.

The source code is available on GitHub:
[https://github.com/emkeyen/postman-to-jmx](https://github.com/emkeyen/postman-to-jmx){:target="_blank"}

### Functionality

- Converts a Postman collection JSON into a JMeter `jmx` test plan.
- Adds support for raw body, form-urlencoded, headers, query params.
- Pulls in both collection-level vars and optional env files as JMeter *User Defined Variables*.
- Handles folders, nested requests, and basic structure in a way JMeter can actually use.

### Usage

Just save the script as `postman2jmx.py`, make it executable, and run it like this:

    :::bash
    chmod +x postman2jmx.py
    ./postman2jmx.py my_collection.json output.jmx -e my_env.json

The `-e` flag is optional. If you pass in an environment JSON exported from Postman, it’ll include those values in the test plan.

### Notes

- A single-thread JMeter test plan (you can tweak that later).
- Every request in Postman becomes a corresponding `HTTPSamplerProxy`.
- Headers and body content (raw or form) are preserved.
- Query params are parsed and added as proper JMeter args.
- Vars show up in the Test Plan as global UDV blocks.
- Variables from your Postman collection and environment will appear as "User Defined Variables" in JMeter. Remember that in JMeter, you reference these variables using ${variable_name}.
- The script handles raw JSON and x-www-form-urlencoded bodies. Other complex body types (like formdata with file uploads) might need manual adjustment in JMeter after conversion.
- Any JS code you have in Postman's pre-request or test scripts won't be converted. You'll need to re-implement that logic in JMeter using JSR223 samplers or other JMeter elements.

Mostly for folks who already use Postman heavily for API development but need to load test without rebuilding everything from scratch in JMeter. You still might need to tweak the resulting `jmx` for more advanced stuff (think auth flows, assertions, etc.), but it saves a bunch of boilerplate work.

    :::python
    #!/usr/bin/env python3
    import json
    import argparse
    import xml.etree.ElementTree as ET
    from xml.dom import minidom
    from urllib.parse import urlparse, parse_qs

    def convert_postman_to_jmx(postman_file, output_file, environment_file=None):
        """
        Converts a Postman collection JSON file to a JMeter JMX file.

        Args:
            postman_file (str): path to the Postman collection JSON file.
            output_file (str): path where the JMeter JMX file will be saved.
            environment_file (str, optional): path to the Postman env JSON file.
                                            if provided, env vars will be added.
        """
        # load Postman collection
        with open(postman_file, 'r') as f:
            collection = json.load(f)

        # load Postman env if provided
        environment_vars = []
        if environment_file:
            try:
                with open(environment_file, 'r') as f_env:
                    environment_data = json.load(f_env)
                    if 'values' in environment_data:
                        # filter for enabled vars from env
                        environment_vars = [v for v in environment_data['values'] if v.get('enabled', True)]
            except FileNotFoundError:
                print(f"Warning: Environment file '{environment_file}' not found. Skipping environment variables.")
            except json.JSONDecodeError:
                print(f"Warning: Could not parse environment file '{environment_file}'. Skipping environment variables.")


        # create JMX root structure
        jmeter_test_plan = ET.Element('jmeterTestPlan', version="1.2", properties="5.0", jmeter="5.2.1")
        hash_tree = ET.SubElement(jmeter_test_plan, 'hashTree')

        # create test plan
        test_plan = ET.SubElement(hash_tree, 'TestPlan', {
            'guiclass': 'TestPlanGui',
            'testclass': 'TestPlan',
            'testname': 'Postman Collection Import',
            'enabled': 'true'
        })
        ET.SubElement(test_plan, 'boolProp', name="TestPlan.functional_mode").text = 'false'
        ET.SubElement(test_plan, 'stringProp', name="TestPlan.comments")
        ET.SubElement(test_plan, 'boolProp', name="TestPlan.serialize_threadgroups").text = 'false'
        ET.SubElement(test_plan, 'stringProp', name="TestPlan.user_define_classpath")

        # add default user defined variables to the test plan (can be empty)
        element_prop = ET.SubElement(test_plan, 'elementProp', {
            'name': 'TestPlan.user_defined_variables',
            'elementType': 'Arguments'
        })
        ET.SubElement(element_prop, 'collectionProp', name="Arguments.arguments")

        hash_tree2 = ET.SubElement(hash_tree, 'hashTree')

        # create thread group
        thread_group = ET.SubElement(hash_tree2, 'ThreadGroup', {
            'guiclass': 'ThreadGroupGui',
            'testclass': 'ThreadGroup',
            'testname': collection.get('info', {}).get('name', 'Postman Requests'),
            'enabled': 'true'
        })

        element_prop = ET.SubElement(thread_group, 'elementProp', {
            'name': 'ThreadGroup.main_controller',
            'elementType': 'LoopController',
            'guiclass': 'LoopControlPanel',
            'testclass': 'LoopController',
            'enabled': 'true'
        })
        ET.SubElement(element_prop, 'boolProp', name="LoopController.continue_forever").text = 'false'
        ET.SubElement(element_prop, 'stringProp', name="LoopController.loops").text = '1'

        ET.SubElement(thread_group, 'stringProp', name="ThreadGroup.num_threads").text = '1'
        ET.SubElement(thread_group, 'stringProp', name="ThreadGroup.ramp_time").text = '1'
        ET.SubElement(thread_group, 'boolProp', name="ThreadGroup.scheduler").text = 'false'
        ET.SubElement(thread_group, 'stringProp', name="ThreadGroup.duration").text = '0'
        ET.SubElement(thread_group, 'stringProp', name="ThreadGroup.delay").text = '0'
        ET.SubElement(thread_group, 'stringProp', name="ThreadGroup.on_sample_error").text = 'continue'
        ET.SubElement(thread_group, 'boolProp', name="ThreadGroup.same_user_on_next_iteration").text = 'true'

        hash_tree3 = ET.SubElement(hash_tree2, 'hashTree')

        # process collection vars
        if 'variable' in collection:
            add_user_defined_variables(collection['variable'], hash_tree3, name="Collection Variables")

        # process env vars
        if environment_vars:
            add_user_defined_variables(environment_vars, hash_tree3, name="Environment Variables")

        # process items recursively (requests and folders)
        if 'item' in collection:
            process_items(collection['item'], hash_tree3)

        # convert to XML string with pretty formatting
        xml_str = ET.tostring(jmeter_test_plan, encoding='utf-8', method='xml')
        dom = minidom.parseString(xml_str)

        # empty elements are properly serialized
        for elem in dom.getElementsByTagName('stringProp'):
            if not elem.firstChild and elem.getAttribute('name') == 'HTTPSampler.port':
                elem.appendChild(dom.createTextNode(''))

        pretty_xml = dom.toprettyxml(indent="    ")

        # remove redundant newlines and spaces added by minidom
        pretty_xml = '\n'.join(line for line in pretty_xml.split('\n') if line.strip())

        with open(output_file, 'w') as f:
            f.write(pretty_xml)

    def add_user_defined_variables(variables, parent_hash_tree, name="User Defined Variables"):
        """
        Adds User Defined Variables to the JMeter JMX structure.

        Args:
            variables (list): A list of dictionaries, where each dictionary represents a variable
                            with 'key' and 'value' (e.g., from Postman collection or environment).
            parent_hash_tree (ET.Element): The parent hashTree element to which the Arguments element will be added.
            name (str): The name to be displayed for this set of User Defined Variables in JMeter.
        """
        if not variables:
            return

        arguments = ET.SubElement(parent_hash_tree, 'Arguments', {
            'guiclass': 'ArgumentsPanel',
            'testclass': 'Arguments',
            'testname': name,
            'enabled': 'true'
        })
        collection_prop = ET.SubElement(arguments, 'collectionProp', name="Arguments.arguments")

        for var in variables:
            # ensure 'key' and 'value' exist before processing
            if 'key' in var and 'value' in var:
                element_prop = ET.SubElement(collection_prop, 'elementProp', {
                    'name': var['key'],
                    'elementType': 'Argument'
                })
                ET.SubElement(element_prop, 'stringProp', name="Argument.name").text = var['key']
                ET.SubElement(element_prop, 'stringProp', name="Argument.value").text = str(var['value'])
                ET.SubElement(element_prop, 'stringProp', name="Argument.metadata").text = '='

        ET.SubElement(parent_hash_tree, 'hashTree') # This hashTree closes the Arguments element

    def process_items(items, parent_element):
        """
        Recursively processes Postman collection items (folders and requests).

        Args:
            items (list): A list of Postman collection items.
            parent_element (ET.Element): The parent XML element to which new elements will be added.
        """
        for item in items:
            if 'item' in item:
                # this is a dir, create a JMeter test fragment or just process its children directly
                process_items(item['item'], parent_element)
            else:
                # request
                process_request(item, parent_element)

    def process_request(item, parent_element):
        """
        Processes a single Postman request and converts it into an HTTPSamplerProxy element.

        Args:
            item (dict): The Postman request item dictionary.
            parent_element (ET.Element): The parent XML element to which the sampler and its hashTree will be added.
        """
        if 'request' not in item:
            return

        request = item['request']

        # init URL and method components with defaults
        method = request.get('method', 'GET')
        domain = ''
        path = ''
        protocol = 'http'
        port = ''

        # create HTTPSamplerProxy
        sampler = ET.SubElement(parent_element, 'HTTPSamplerProxy', {
            'guiclass': 'HttpTestSampleGui',
            'testclass': 'HTTPSamplerProxy',
            'testname': item.get('name', 'Unnamed Request'),
            'enabled': 'true'
        })

        # process request body
        if 'body' in request:
            body = request['body']
            if body.get('mode') == 'raw' and 'raw' in body and body['raw']:
                ET.SubElement(sampler, 'boolProp', name="HTTPSampler.postBodyRaw").text = 'true'
                element_prop = ET.SubElement(sampler, 'elementProp', {
                    'name': 'HTTPsampler.Arguments',
                    'elementType': 'Arguments',
                    'guiclass': 'HTTPArgumentsPanel',
                    'testclass': 'Arguments',
                    'enabled': 'true'
                })
                collection_prop = ET.SubElement(element_prop, 'collectionProp', name="Arguments.arguments")

                arg_element = ET.SubElement(collection_prop, 'elementProp', {
                    'name': '', # name is empty for raw body
                    'elementType': 'HTTPArgument'
                })
                ET.SubElement(arg_element, 'boolProp', name="HTTPArgument.always_encode").text = 'false'
                ET.SubElement(arg_element, 'stringProp', name="Argument.value").text = body['raw']
                ET.SubElement(arg_element, 'stringProp', name="Argument.metadata").text = '='
            elif body.get('mode') == 'urlencoded' and 'urlencoded' in body and body['urlencoded']:
                # for x-www-form-urlencoded, JMeter handles params as args
                element_prop = ET.SubElement(sampler, 'elementProp', {
                    'name': 'HTTPsampler.Arguments',
                    'elementType': 'Arguments',
                    'guiclass': 'HTTPArgumentsPanel',
                    'testclass': 'Arguments',
                    'enabled': 'true'
                })
                collection_prop = ET.SubElement(element_prop, 'collectionProp', name="Arguments.arguments")

                for param in body['urlencoded']:
                    arg_element = ET.SubElement(collection_prop, 'elementProp', {
                        'name': param.get('key', ''),
                        'elementType': 'HTTPArgument'
                    })
                    ET.SubElement(arg_element, 'boolProp', name="HTTPArgument.always_encode").text = 'false'
                    ET.SubElement(arg_element, 'stringProp', name="Argument.value").text = param.get('value', '')
                    ET.SubElement(arg_element, 'stringProp', name="Argument.metadata").text = '='
                    ET.SubElement(arg_element, 'boolProp', name="HTTPArgument.use_equals").text = 'true'
                    ET.SubElement(arg_element, 'stringProp', name="Argument.name").text = param.get('key', '')
            else:
                # handle other body types or empty body
                element_prop = ET.SubElement(sampler, 'elementProp', {
                    'name': 'HTTPsampler.Arguments',
                    'elementType': 'Arguments',
                    'guiclass': 'HTTPArgumentsPanel',
                    'testclass': 'Arguments',
                    'enabled': 'true'
                })
                ET.SubElement(element_prop, 'collectionProp', name="Arguments.arguments")
        else:
            # no body in request
            element_prop = ET.SubElement(sampler, 'elementProp', {
                'name': 'HTTPsampler.Arguments',
                'elementType': 'Arguments',
                'guiclass': 'HTTPArgumentsPanel',
                'testclass': 'Arguments',
                'enabled': 'true'
            })
            ET.SubElement(element_prop, 'collectionProp', name="Arguments.arguments")


        # set common sampler props (bool props)
        ET.SubElement(sampler, 'boolProp', name="HTTPSampler.auto_redirects").text = 'false'
        ET.SubElement(sampler, 'boolProp', name="HTTPSampler.follow_redirects").text = 'true'
        ET.SubElement(sampler, 'boolProp', name="HTTPSampler.use_keepalive").text = 'true'
        ET.SubElement(sampler, 'boolProp', name="HTTPSampler.monitor").text = 'false'
        ET.SubElement(sampler, 'boolProp', name="HTTPSampler.DO_MULTIPART_POST").text = 'false' # may need adjustment for multipart/form-data
        ET.SubElement(sampler, 'stringProp', name="HTTPSampler.embedded_url_re")
        ET.SubElement(sampler, 'stringProp', name="HTTPSampler.contentEncoding")


        # process URL (logic to determine domain, path, protocol, port)
        if 'url' in request:
            url_data = request['url']
            if isinstance(url_data, str):
                parsed_url = urlparse(url_data)
                domain = parsed_url.hostname or ''
                path = parsed_url.path or ''
                protocol = parsed_url.scheme or 'http'
                port = str(parsed_url.port) if parsed_url.port else '' # Ensure port is string, even if None

                # handle query paramsfrom URL string
                if parsed_url.query:
                    query_args_element_prop = sampler.find("./elementProp[@name='HTTPsampler.Arguments']")
                    if query_args_element_prop is None: # Create if not already present from body
                        query_args_element_prop = ET.SubElement(sampler, 'elementProp', {
                            'name': 'HTTPsampler.Arguments',
                            'elementType': 'Arguments',
                            'guiclass': 'HTTPArgumentsPanel',
                            'testclass': 'Arguments',
                            'enabled': 'true'
                        })
                        ET.SubElement(query_args_element_prop, 'collectionProp', name="Arguments.arguments")

                    query_collection_prop = query_args_element_prop.find("collectionProp[@name='Arguments.arguments']")
                    
                    query_params = parse_qs(parsed_url.query)
                    for key, values in query_params.items():
                        for value in values:
                            arg_element = ET.SubElement(query_collection_prop, 'elementProp', {
                                'name': key,
                                'elementType': 'HTTPArgument'
                            })
                            ET.SubElement(arg_element, 'boolProp', name="HTTPArgument.always_encode").text = 'true'
                            ET.SubElement(arg_element, 'stringProp', name="Argument.value").text = value
                            ET.SubElement(arg_element, 'stringProp', name="Argument.metadata").text = '='
                            ET.SubElement(arg_element, 'boolProp', name="HTTPArgument.use_equals").text = 'true'
                            ET.SubElement(arg_element, 'stringProp', name="Argument.name").text = key

            elif isinstance(url_data, dict):
                # Postman URL object structure
                domain = '.'.join(url_data.get('host', ['localhost']))
                path_parts = url_data.get('path', [])
                if isinstance(path_parts, list):
                    path = '/' + '/'.join(path_parts)
                else: # assume it's a string
                    path = path_parts
                protocol = url_data.get('protocol', 'http')
                if protocol.endswith(':'): # Remove trailing colon if present
                    protocol = protocol[:-1]
                port = str(url_data.get('port', '')) # Ensure port is string, even if empty

                # process query params if present in URL object
                if 'query' in url_data and url_data['query']:
                    args_element = sampler.find("./elementProp[@name='HTTPsampler.Arguments']/collectionProp[@name='Arguments.arguments']")
                    if args_element is None:
                        element_prop = ET.SubElement(sampler, 'elementProp', {
                            'name': 'HTTPsampler.Arguments',
                            'elementType': 'Arguments',
                            'guiclass': 'HTTPArgumentsPanel',
                            'testclass': 'Arguments',
                            'enabled': 'true'
                        })
                        args_element = ET.SubElement(element_prop, 'collectionProp', name="Arguments.arguments")

                    for param in url_data['query']:
                        if 'key' in param and 'value' in param:
                            arg_element = ET.SubElement(args_element, 'elementProp', {
                                'name': param.get('key', ''),
                                'elementType': 'HTTPArgument'
                            })
                            ET.SubElement(arg_element, 'boolProp', name="HTTPArgument.always_encode").text = 'true'
                            ET.SubElement(arg_element, 'stringProp', name="Argument.value").text = param.get('value', '')
                            ET.SubElement(arg_element, 'stringProp', name="Argument.metadata").text = '='
                            ET.SubElement(arg_element, 'boolProp', name="HTTPArgument.use_equals").text = 'true'
                            ET.SubElement(arg_element, 'stringProp', name="Argument.name").text = param.get('key', '')

        # create all primary sampler props using the extracted values
        ET.SubElement(sampler, 'stringProp', name="HTTPSampler.method").text = method
        ET.SubElement(sampler, 'stringProp', name="HTTPSampler.domain").text = domain
        ET.SubElement(sampler, 'stringProp', name="HTTPSampler.path").text = path
        ET.SubElement(sampler, 'stringProp', name="HTTPSampler.protocol").text = protocol
        port_prop = ET.SubElement(sampler, 'stringProp', name="HTTPSampler.port")
        port_prop.text = port or ''  # This handles both None and empty string


        # create hashTree for sampler
        sampler_hash_tree = ET.SubElement(parent_element, 'hashTree')

        # process headers
        if 'header' in request and request['header']:
            header_manager = ET.SubElement(sampler_hash_tree, 'HeaderManager', {
                'guiclass': 'HeaderPanel',
                'testclass': 'HeaderManager',
                'testname': 'HTTP Header Manager',
                'enabled': 'true'
            })
            collection_prop = ET.SubElement(header_manager, 'collectionProp', name="HeaderManager.headers")

            for header in request['header']:
                element_prop = ET.SubElement(collection_prop, 'elementProp', {
                    'name': '',
                    'elementType': 'Header'
                })
                ET.SubElement(element_prop, 'stringProp', name="Header.name").text = header.get('key', '')
                ET.SubElement(element_prop, 'stringProp', name="Header.value").text = header.get('value', '')

            ET.SubElement(sampler_hash_tree, 'hashTree')

        # process URL vars (path vars in Postman terminology)
        if 'url' in request and isinstance(request['url'], dict) and 'variable' in request['url'] and request['url']['variable']:
            add_user_defined_variables(request['url']['variable'], sampler_hash_tree, name="URL Path Variables")


    def main():
        """
        Main func to parse args and init the conversion
        """
        parser = argparse.ArgumentParser(description='Convert Postman collection to JMeter JMX')
        parser.add_argument('input', help='Postman collection JSON file')
        parser.add_argument('output', help='Output JMX file')
        parser.add_argument('-e', '--environment', help='Postman environment JSON file (optional)', default=None)
        args = parser.parse_args()

        convert_postman_to_jmx(args.input, args.output, args.environment)
        print(f"Successfully converted {args.input} to {args.output}")

    if __name__ == '__main__':
        main()
